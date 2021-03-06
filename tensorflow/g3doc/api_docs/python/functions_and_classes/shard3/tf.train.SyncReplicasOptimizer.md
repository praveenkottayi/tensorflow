Class to synchronize, aggregate gradients and pass them to the optimizer.

In a typical asynchronous training environment, it's common to have some
stale gradients. For example, with a N-replica asynchronous training,
gradients will be applied to the variables N times independently. Depending
on each replica's training speed, some gradients might be calculated from
copies of the variable from several steps back (N-1 steps on average). This
optimizer avoids stale gradients by collecting gradients from all replicas,
summing them, then applying them to the variables in one shot, after
which replicas can fetch the new variables and continue.

The following queues are created:
<empty line>
* N `gradient` queues, one per variable to train. Gradients are pushed to
  these queues and the chief worker will dequeue_many and then sum them
  before applying to variables.
* 1 `token` queue where the optimizer pushes the new global_step value after
  all gradients have been applied.

The following variables are created:
* N `local_step`, one per replica. Compared against global step to check for
  staleness of the gradients.

This adds nodes to the graph to collect gradients and pause the trainers until
variables are updated.
For the PS:
<empty line>
1. A queue is created for each variable, and each replica now pushes the
  gradients into the queue instead of directly applying them to the
  variables.
2. For each gradient_queue, pop and sum the gradients once enough
  replicas (replicas_to_aggregate) have pushed gradients to the queue.
3. Apply the aggregated gradients to the variables.
4. Only after all variables have been updated, increment the global step.
5. Only after step 4, clear all the gradients in the queues as they are
  stale now (could happen when replicas are restarted and push to the queues
  multiple times, or from the backup replicas).
6. Only after step 5, pushes `global_step` in the `token_queue`, once for
  each worker replica. The workers can now fetch it to its local_step variable
  and start the next batch.

For the replicas:
<empty line>
1. Start a step: fetch variables and compute gradients.
2. Once the gradients have been computed, push them into `gradient_queue` only
  if local_step equals global_step, otherwise the gradients are just dropped.
  This avoids stale gradients.
3. After pushing all the gradients, dequeue an updated value of global_step
  from the token queue and record that step to its local_step variable. Note
  that this is effectively a barrier.
4. Start the next batch.

### Usage

```python
# Create any optimizer to update the variables, say a simple SGD:
opt = GradientDescentOptimizer(learning_rate=0.1)

# Wrap the optimizer with sync_replicas_optimizer with 50 replicas: at each
# step the optimizer collects 50 gradients before applying to variables.
opt = tf.SyncReplicasOptimizer(opt, replicas_to_aggregate=50,
          replica_id=task_id, total_num_replicas=50)
# Note that if you want to have 2 backup replicas, you can change
# total_num_replicas=52 and make sure this number matches how many physical
# replicas you started in your job.

# Some models have startup_delays to help stabilize the model but when using
# sync_replicas training, set it to 0.

# Now you can call `minimize()` or `compute_gradients()` and
# `apply_gradients()` normally
grads = opt.minimize(total_loss, global_step=self.global_step)


# You can now call get_init_tokens_op() and get_chief_queue_runner().
# Note that get_init_tokens_op() must be called before creating session
# because it modifies the graph.
init_token_op = opt.get_init_tokens_op()
chief_queue_runner = opt.get_chief_queue_runner()
```

In the training program, every worker will run the train_op as if not
synchronized. But one worker (usually the chief) will need to execute the
chief_queue_runner and get_init_tokens_op generated from this optimizer.

```python
# After the session is created by the Supervisor and before the main while
# loop:
if is_chief and FLAGS.sync_replicas:
  sv.start_queue_runners(sess, [chief_queue_runner])
  # Insert initial tokens to the queue.
  sess.run(init_token_op)
```

- - -

#### `tf.train.SyncReplicasOptimizer.__init__(opt, replicas_to_aggregate, variable_averages=None, variables_to_average=None, replica_id=None, total_num_replicas=0, use_locking=False, name='sync_replicas')` {#SyncReplicasOptimizer.__init__}

Construct a sync_replicas optimizer.

##### Args:


*  <b>`opt`</b>: The actual optimizer that will be used to compute and apply the
    gradients. Must be one of the Optimizer classes.
*  <b>`replicas_to_aggregate`</b>: number of replicas to aggregate for each variable
    update.
*  <b>`variable_averages`</b>: Optional `ExponentialMovingAverage` object, used to
    maintain moving averages for the variables passed in
    `variables_to_average`.
*  <b>`variables_to_average`</b>: a list of variables that need to be averaged. Only
    needed if variable_averages is passed in.
*  <b>`replica_id`</b>: This is the task/worker/replica ID. Needed as index to access
    local_steps to check staleness. Must be in the interval:
    [0, total_num_replicas)
*  <b>`total_num_replicas`</b>: Total number of tasks/workers/replicas, could be
    different from replicas_to_aggregate.
    If total_num_replicas > replicas_to_aggregate: it is backup_replicas +
    replicas_to_aggregate.
    If total_num_replicas < replicas_to_aggregate: Replicas compute
    multiple batches per update to variables.
*  <b>`use_locking`</b>: If True use locks for update operation.
*  <b>`name`</b>: string. Optional name of the returned operation.


- - -

#### `tf.train.SyncReplicasOptimizer.compute_gradients(*args, **kwargs)` {#SyncReplicasOptimizer.compute_gradients}

Compute gradients of "loss" for the variables in "var_list".

This simply wraps the compute_gradients() from the real optimizer. The
gradients will be aggregated in the apply_gradients() so that user can
modify the gradients like clipping with per replica global norm if needed.
The global norm with aggregated gradients can be bad as one replica's huge
gradients can hurt the gradients from other replicas.

##### Args:


*  <b>`*args`</b>: Arguments for compute_gradients().
*  <b>`**kwargs`</b>: Keyword arguments for compute_gradients().

##### Returns:

  A list of (gradient, variable) pairs.


- - -

#### `tf.train.SyncReplicasOptimizer.apply_gradients(grads_and_vars, global_step=None, name=None)` {#SyncReplicasOptimizer.apply_gradients}

Apply gradients to variables.

This contains most of the synchronization implementation and also wraps the
apply_gradients() from the real optimizer.

##### Args:


*  <b>`grads_and_vars`</b>: List of (gradient, variable) pairs as returned by
    compute_gradients().
*  <b>`global_step`</b>: Optional Variable to increment by one after the
    variables have been updated.
*  <b>`name`</b>: Optional name for the returned operation.  Default to the
    name passed to the Optimizer constructor.

##### Returns:


*  <b>`train_op`</b>: The op to dequeue a token so the replicas can exit this batch
  and start the next one. This is executed by each replica.

##### Raises:


*  <b>`ValueError`</b>: If the grads_and_vars is empty.
*  <b>`ValueError`</b>: If global step is not provided, the staleness cannot be
    checked.


- - -

#### `tf.train.SyncReplicasOptimizer.get_chief_queue_runner()` {#SyncReplicasOptimizer.get_chief_queue_runner}

Returns the QueueRunner for the chief to execute.

This includes the operations to synchronize replicas: aggregate gradients,
apply to variables, increment global step, insert tokens to token queue.

Note that this can only be called after calling apply_gradients() which
actually generates this queuerunner.

##### Returns:

  A `QueueRunner` for chief to execute.

##### Raises:


*  <b>`ValueError`</b>: If this is called before apply_gradients().


- - -

#### `tf.train.SyncReplicasOptimizer.get_init_tokens_op(num_tokens=-1)` {#SyncReplicasOptimizer.get_init_tokens_op}

Returns the op to fill the sync_token_queue with the tokens.

This is supposed to be executed in the beginning of the chief/sync thread
so that even if the total_num_replicas is less than replicas_to_aggregate,
the model can still proceed as the replicas can compute multiple steps per
variable update. Make sure:
`num_tokens >= replicas_to_aggregate - total_num_replicas`.

##### Args:


*  <b>`num_tokens`</b>: Number of tokens to add to the queue.

##### Returns:

  An op for the chief/sync replica to fill the token queue.

##### Raises:


*  <b>`ValueError`</b>: If this is called before apply_gradients().
*  <b>`ValueError`</b>: If num_tokens are smaller than replicas_to_aggregate -
    total_num_replicas.



#### Other Methods
- - -

#### `tf.train.SyncReplicasOptimizer.get_clean_up_op()` {#SyncReplicasOptimizer.get_clean_up_op}

Returns the clean up op for the chief to execute before exit.

This includes the operation to abort the device with the token queue so all
other replicas can also restart. This can avoid potential hang when chief
restarts.

Note that this can only be called after calling apply_gradients().

##### Returns:

  A clean_up_op for chief to execute before exits.

##### Raises:


*  <b>`ValueError`</b>: If this is called before apply_gradients().


- - -

#### `tf.train.SyncReplicasOptimizer.get_slot(*args, **kwargs)` {#SyncReplicasOptimizer.get_slot}

Return a slot named "name" created for "var" by the Optimizer.

This simply wraps the get_slot() from the actual optimizer.

##### Args:


*  <b>`*args`</b>: Arguments for get_slot().
*  <b>`**kwargs`</b>: Keyword arguments for get_slot().

##### Returns:

  The `Variable` for the slot if it was created, `None` otherwise.


- - -

#### `tf.train.SyncReplicasOptimizer.get_slot_names(*args, **kwargs)` {#SyncReplicasOptimizer.get_slot_names}

Return a list of the names of slots created by the `Optimizer`.

This simply wraps the get_slot_names() from the actual optimizer.

##### Args:


*  <b>`*args`</b>: Arguments for get_slot().
*  <b>`**kwargs`</b>: Keyword arguments for get_slot().

##### Returns:

  A list of strings.


