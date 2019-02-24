# Checkpoint
Flink 容错机制的核心就是持续创建分布式数据流及其状态的一致快照。Flink的checkpoint 是通过分布式快照实现的,所以在flink中这两个词是一个意思。
- checkpoint用来保证任务的错误恢复。任务失败可以从最新的checkpoint恢复。
- checkpoint机制需要一个可靠的可以回放数据的数据源(kafka,RabbitMQ,HDFS...)和一个存放state的持久存储（HDFS,S3...）。

## 1，checkpointConfig

- 通过调用StreamExecutionEnvironment.enableCheckpointing(internal，mode)启用checkpoint。 internal默认是-1，表示checkpoint不开启，mode默认是EXACTLY_ONCE模式。

- 可设置checkpoint timeout,超过这个时间checkpoint没有成功，checkpoint终止。默认10分钟。
- 可设置checkpoint失败任务是否也失败，默认是true。
- 可设置同时进行的checkpoint数量，默认为1。

## 2，barrier
- 将barrier插入到数据流中，作为数据流的一部分和数据一起向下流动。Barrier 不会干扰正常数据，数据流严格有序。一个 barrier 把数据流分割成两部分：一部分进入到当前快照，另一部分进入下一个快照。每一个 barrier 都带有快照 ID，并且 barrier 之前的数据都进入了此快照。Barrier 不会干扰数据流处理，所以非常轻量。多个不同快照的多个 barrier 会在流中同时出现，即多个快照可能同时创建。
- Barrier 在数据源端插入，当 snapshot n 的 barrier 插入后，系统会记录当前 snapshot 位置值 n (用Sn表示)。例如，在 Apache Kafka 中，这个变量表示某个分区中最后一条数据的偏移量。这个位置值 Sn 会被发送到一个称为 checkpoint coordinator 的模块。
- 然后 barrier 继续往下流动，当一个 operator 从其输入流接收到所有标识 snapshot n 的 barrier 时，它会向其所有输出流插入一个标识 snapshot n 的 barrier。当 sink operator （DAG 流的终点）从其输入流接收到所有 barrier n 时，它向 the checkpoint coordinator 确认 snapshot n 已完成。当所有 sink 都确认了这个快照，快照就被标识为完成。

## 3，如何触发checkpoint？
- 在生成EcecutionGraph过程中会注册状态监听器CheckpointCoordinatorDeActivator，负责监听job状态，job变为运行状态时，执行startCheckpointScheduler方法定时执行ScheduledTrigger，ScheduledTrigger中执行的是triggerCheckpoint方法。
- 在进行一些条件检查之后，首先会构造出一个PendingCheckpoint实例，然后再放到队列里，只有当jobmanager收到SinkTask发来的checkpoint保存成功的消息后，这个PendingCheckpoint才会变成CompletedCheckpoint，这才代表一次checkpoint保存操作真正的完成了。
```
final PendingCheckpoint checkpoint = 
new PendingCheckpoint(
    job,
    checkpointID,
    timestamp,
    ackTasks,
    props,
    checkpointStorageLocation,
    executor);

if (statsTracker != null) {
    PendingCheckpointStats callback = statsTracker.reportPendingCheckpoint(
        checkpointID,
        timestamp,
        props);

    checkpoint.setStatsCallback(callback);
}
```
再往下看，最重要的方法是向executions发送保存checkpoint的消息，通知taskmanager进行本地的checkpoint保存。
```
// send the messages to the tasks that trigger their checkpoint
for (Execution execution: executions) {
    execution.triggerCheckpoint(checkpointID, timestamp, checkpointOptions);
}
```
下面看看executions从哪来,下面这段代码在PendingCheckpoint生成之前执行：
```
Execution[] executions = new Execution[tasksToTrigger.length];
for (int i = 0; i < tasksToTrigger.length; i++) {
    Execution ee = tasksToTrigger[i].getCurrentExecutionAttempt();
    if (ee == null) {
        LOG.info("Checkpoint triggering task {} of job {} is not being executed at the moment. Aborting checkpoint.",
                tasksToTrigger[i].getTaskNameWithSubtaskIndex(),
                job);
        return new CheckpointTriggerResult(CheckpointDeclineReason.NOT_ALL_REQUIRED_TASKS_RUNNING);
    } else if (ee.getState() == ExecutionState.RUNNING) {
        executions[i] = ee;
    } else {
        LOG.info("Checkpoint triggering task {} of job {} is not in state {} but {} instead. Aborting checkpoint.",
                tasksToTrigger[i].getTaskNameWithSubtaskIndex(),
                job,
                ExecutionState.RUNNING,
                ee.getState());
        return new CheckpointTriggerResult(CheckpointDeclineReason.NOT_ALL_REQUIRED_TASKS_RUNNING);
    }
}
```
可以看到execution中的元素都是从tasksToTrigger中获得，那t
asksToTrigger是什么呢？可以追溯到ExecutionGraph中：
```
ExecutionVertex[] tasksToTrigger = collectExecutionVertices(verticesToTrigger);
```
看了下collectExecutionVertices方法，并没有说明来源，继续看verticesToTrigger从哪里来，追溯到StreamingJobGraphGenerator中：
```
for (JobVertex vertex : jobVertices.values()) {
			if (vertex.isInputVertex()) {
				triggerVertices.add(vertex.getID());
			}
			commitVertices.add(vertex.getID());
			ackVertices.add(vertex.getID());
		}
```
```
public boolean isInputVertex() {
		return this.inputs.isEmpty();
	}
```
就是说triggerVertices是那些没有输入的节点，即数据源。
- 综上，配置了checkpoint，job状态监听器在监听到状态为running时，会开启定时器，定时向TaskManager发送TriggerCheckpoint消息。

下面看看TaskManager收到TriggerCheckpoint消息之后如何处理
。
```
case message: TriggerCheckpoint =>
    val taskExecutionId = message.getTaskExecutionId
    val checkpointId = message.getCheckpointId
    val timestamp = message.getTimestamp
    val checkpointOptions = message.getCheckpointOptions

    log.debug(s"Receiver TriggerCheckpoint $checkpointId@$timestamp for $taskExecutionId.")

    val task = runningTasks.get(taskExecutionId)
    if (task != null) {
        task.triggerCheckpointBarrier(checkpointId, timestamp, checkpointOptions)
    } else {
        log.debug(s"TaskManager received a checkpoint request for unknown task $taskExecutionId.")
    }
```
关键在triggerCheckpointBarrier方法，方法中最核心的是这个方法：
```
boolean success = invokable.triggerCheckpoint(checkpointMetaData, checkpointOptions);
```
因为之前已经知道消息是发往数据源task的，所以在所有triggerCheckpoint实现中看下SourceStreamTask的,最终追溯到StreamTask的performCheckpoint方法：
```
private boolean performCheckpoint(
        CheckpointMetaData checkpointMetaData,
        CheckpointOptions checkpointOptions,
        CheckpointMetrics checkpointMetrics) throws Exception {
    synchronized (lock) {
        if (isRunning) {
            
            // 第一步：为checkpoint做一些准备工作，默认什么都不做
            operatorChain.prepareSnapshotPreBarrier(checkpointMetaData.getCheckpointId());

            // 第二步：向下游广播发送checkpoint barrier
            operatorChain.broadcastCheckpointBarrier(
                    checkpointMetaData.getCheckpointId(),
                    checkpointMetaData.getTimestamp(),
                    checkpointOptions);

            // 第三步：异步存储状态快照
            checkpointState(checkpointMetaData, checkpointOptions, checkpointMetrics);
            return true;
        }
        else {
            。。。任务不是运行状态，不触发checkpoint，向下游广播取消checkpoint的消息。
            return false;
        }
    }
}
```
其实下游每个task收到checkpoint barrier都会执行上述操作。

### 3.2 异步储存快照
进入checkpointState方法，追溯至executeCheckpointing方法：
```
public void executeCheckpointing() throws Exception {
        startSyncPartNano = System.nanoTime();
        try {
            for (StreamOperator<?> op : allOperators) {
                checkpointStreamOperator(op);
            }

            startAsyncPartNano = System.nanoTime();

            checkpointMetrics.setSyncDurationMillis((startAsyncPartNano - startSyncPartNano) / 1_000_000);

            AsyncCheckpointRunnable asyncCheckpointRunnable = new AsyncCheckpointRunnable(
                owner,
                operatorSnapshotsInProgress,
                checkpointMetaData,
                checkpointMetrics,
                startAsyncPartNano);

            owner.cancelables.registerCloseable(asyncCheckpointRunnable);
            owner.asyncOperationsThreadPool.submit(asyncCheckpointRunnable);
        } catch (Exception ex) {
            ...
        }
    }
}
```
- 对task中所有operator执行checkpointStreamOperator方法，这个方法是opertor存储快照的地方。
```
private void checkpointStreamOperator(StreamOperator<?> op) throws Exception {
			if (null != op) {
				OperatorSnapshotFutures snapshotInProgress = op.snapshotState(
						checkpointMetaData.getCheckpointId(),
						checkpointMetaData.getTimestamp(),
						checkpointOptions,
						storageLocation);
				operatorSnapshotsInProgress.put(op.getOperatorID(), snapshotInProgress);
			}
		}
```
- 接着看snapshotState方法：
```
public final OperatorSnapshotFutures snapshotState(long checkpointId, long timestamp, CheckpointOptions checkpointOptions,
			CheckpointStreamFactory factory) throws Exception {

		KeyGroupRange keyGroupRange = null != keyedStateBackend ?
				keyedStateBackend.getKeyGroupRange() : KeyGroupRange.EMPTY_KEY_GROUP_RANGE;

		OperatorSnapshotFutures snapshotInProgress = new OperatorSnapshotFutures();

		try (StateSnapshotContextSynchronousImpl snapshotContext = new StateSnapshotContextSynchronousImpl(
				checkpointId,
				timestamp,
				factory,
				keyGroupRange,
				getContainingTask().getCancelables())) {

			snapshotState(snapshotContext);
            
			snapshotInProgress.setKeyedStateRawFuture(snapshotContext.getKeyedStateStreamFuture());
			snapshotInProgress.setOperatorStateRawFuture(snapshotContext.getOperatorStateStreamFuture());
            //这里调用operator的snapshot异步方法
			if (null != operatorStateBackend) {
				snapshotInProgress.setOperatorStateManagedFuture(
					operatorStateBackend.snapshot(checkpointId, timestamp, factory, checkpointOptions));
			}

			if (null != keyedStateBackend) {
				snapshotInProgress.setKeyedStateManagedFuture(
					keyedStateBackend.snapshot(checkpointId, timestamp, factory, checkpointOptions));
			}
		} 
		return snapshotInProgress;
	}
```
operatorStateBackend是保存状态的介质，Flink中提供了三种不同的存储介质:heap，hdfs，rockdb。将Future加入到snapshotInProgress中，全部完成后执行AsyncCheckpointRunnable。
- AsyncCheckpointRunnable线程的run方法：
```
public void run() {
    FileSystemSafetyNet.initializeSafetyNetForThread();
    try {

        TaskStateSnapshot jobManagerTaskOperatorSubtaskStates =
            new TaskStateSnapshot(operatorSnapshotsInProgress.size());

        TaskStateSnapshot localTaskOperatorSubtaskStates =
            new TaskStateSnapshot(operatorSnapshotsInProgress.size());
        
        for (Map.Entry<OperatorID, OperatorSnapshotFutures> entry : operatorSnapshotsInProgress.entrySet()) {

            OperatorID operatorID = entry.getKey();
            OperatorSnapshotFutures snapshotInProgress = entry.getValue();

            // finalize the async part of all by executing all snapshot runnables
            OperatorSnapshotFinalizer finalizedSnapshots =
                new OperatorSnapshotFinalizer(snapshotInProgress);

            jobManagerTaskOperatorSubtaskStates.putSubtaskStateByOperatorID(
                operatorID,
                finalizedSnapshots.getJobManagerOwnedState());

            localTaskOperatorSubtaskStates.putSubtaskStateByOperatorID(
                operatorID,
                finalizedSnapshots.getTaskLocalState());
        }

        final long asyncEndNanos = System.nanoTime();
        final long asyncDurationMillis = (asyncEndNanos - asyncStartNanos) / 1_000_000L;
        // checkpoint 完成时间
        checkpointMetrics.setAsyncDurationMillis(asyncDurationMillis);
        // checkpoint完成后，通知jobmanager checkpoint完成
        if (asyncCheckpointState.compareAndSet(CheckpointingOperation.AsyncCheckpointState.RUNNING,
            CheckpointingOperation.AsyncCheckpointState.COMPLETED)) {
            reportCompletedSnapshotStates(
                jobManagerTaskOperatorSubtaskStates,
                localTaskOperatorSubtaskStates,
                asyncDurationMillis);
        } else {
            LOG....
        }
    }
}
```

- reportCompletedSnapshotStates方法，在存储完state之后会通过CheckpointResponder发送消息给jobmanager告知checkpoint完成。jobmanager在所有task都通知了同一个id的checkpoint完成之后才会算这个checkpoint完成。

