## Jobmanager处理SubmitJob

先将submitJob的主要步骤总结写在开头，然后一步步分析。

- 1，通过JobGraph生成ExecutionGraph;
- 2，恢复状态CheckpointedState，或者Savepoint;
- 3，提交Execution给Scheduler进行调度；
  - 3.1 获取ExecutionGraph中所有vertice，并为其分配slot资源;
  - 3.2 通知TaskManager，将每个vertice部署在分配好的资源中。

下面一步一步分析：

### 1，**通过JobGraph生成ExecutionGraph**

- jobManager接收到SubmitJob消息后，生成了一个jobInfo对象装载job信息，然后调用submitJob方法。

```
case SubmitJob(jobGraph, listeningBehaviour) =>
      val client = sender()
      val jobInfo = new JobInfo(client, listeningBehaviour, System.currentTimeMillis(),
        jobGraph.getSessionTimeout)
      submitJob(jobGraph, jobInfo)
```

- 深入submitJob方法，首先判断jobGraph是否为空，如果为空，返回JobResultFailure消息；

```
if (jobGraph == null) {
      jobInfo.notifyClients(
        decorateMessage(JobResultFailure(
          new SerializedThrowable(
            new JobSubmissionException(null, "JobGraph must not be null.")))))
}
```

- 接着向类库缓存管理器注册该Job相关的库文件、类路径；必须确保该步骤在第一步执行，因为后续产生任何异常可以确保上传的类库和Jar等成功从类库缓存管理器移除。

```
libraryCacheManager.registerJob(
            jobGraph.getJobID, jobGraph.getUserJarBlobKeys, jobGraph.getClasspaths)
```

- 接下来是获得用户代码的类加载器classLoader以及发生失败时的重启策略restartStrategy；

```
val userCodeLoader = libraryCacheManager.getClassLoader(jobGraph.getJobID)
...
val restartStrategy =
    Option(jobGraph.getSerializedExecutionConfig()
        .deserializeValue(userCodeLoader)
        .getRestartStrategy())
        .map(RestartStrategyFactory.createRestartStrategy)
        .filter(p => p != null) match {
        case Some(strategy) => strategy
        case None => restartStrategyFactory.createRestartStrategy()
	}
```

- 接着，获取ExecutionGraph对象的实例。首先尝试从缓存中查找，如果缓存中存在则直接返回，否则直接创建然后加入缓存；

```
val registerNewGraph = currentJobs.get(jobGraph.getJobID) match {
    case Some((graph, currentJobInfo)) =>
    	executionGraph = graph
    	currentJobInfo.setLastActive()
    	false
    case None =>
    	true
}

val allocationTimeout: Long = flinkConfiguration.getLong(
	JobManagerOptions.SLOT_REQUEST_TIMEOUT)

val resultPartitionLocationTrackerProxy: ResultPartitionLocationTrackerProxy =
	new ResultPartitionLocationTrackerProxy(flinkConfiguration)

executionGraph = ExecutionGraphBuilder.buildGraph(
    executionGraph,
    jobGraph,
    flinkConfiguration,
    futureExecutor,
    ioExecutor,
    scheduler,
    userCodeLoader,
    checkpointRecoveryFactory,
    Time.of(timeout.length, timeout.unit),
    restartStrategy,
    jobMetrics,
    numSlots,
    blobServer,
    resultPartitionLocationTrackerProxy,
    Time.milliseconds(allocationTimeout),
    log.logger)
...
//加入缓存
if (registerNewGraph) {
   currentJobs.put(jobGraph.getJobID, (executionGraph, jobInfo))
}
```

- 接着根据配置生成带有graphManagerPlugin的graphManager（**后面需要用到这个**），和operationLogManager；

```
val conf = new Configuration(jobGraph.getJobConfiguration)
conf.addAll(jobGraph.getSchedulingConfiguration)
val graphManagerPlugin = GraphManagerPluginFactory.createGraphManagerPlugin(
	jobGraph.getSchedulingConfiguration, userCodeLoader)
val operationLogManager = new OperationLogManager(
	OperationLogStoreLoader.loadOperationLogStore(jobGraph.getJobID(), conf))
val graphManager =
	new GraphManager(graphManagerPlugin, null, operationLogManager, executionGraph)
graphManager.open(jobGraph, new SchedulingConfig(conf, userCodeLoader))
executionGraph.setGraphManager(graphManager)
operationLogManager.start()
```

- 注册Job状态变化的事件回调给jobmanager自己;

```
executionGraph.registerJobStatusListener(
          new StatusListenerMessenger(self, leaderSessionID.orNull))
```

- 注册整个job状态变化事件回调和单个task状态变化回调给client；

```
jobInfo.clients foreach {
    // the sender wants to be notified about state changes
    case (client, ListeningBehaviour.EXECUTION_RESULT_AND_STATE_CHANGES) =>
    	val listener  = new StatusListenerMessenger(client, leaderSessionID.orNull)
    	executionGraph.registerExecutionListener(listener)
    	executionGraph.registerJobStatusListener(listener)
    case _ => // do nothing
}
```

- 生成executionGraph细节在另一个文章中分析，获取executionGraph之后，如何提交job？继续看；

### 2，**恢复状态CheckpointedState，或者Savepoint**

- 如果是恢复的job，从最新的checkpoint中恢复；

```
if (isRecovery) {
    // this is a recovery of a master failure (this master takes over)
    executionGraph.restoreLatestCheckpointedState(false, false)
}
```

- 或者获取savepoint配置，如果配置了savepoint，便从savepoint中恢复；

```
val savepointSettings = jobGraph.getSavepointRestoreSettings
if (savepointSettings.restoreSavepoint()) {
    try {
        val savepointPath = savepointSettings.getRestorePath()
        val allowNonRestored = savepointSettings.allowNonRestoredState()
        val resumeFromLatestCheckpoint = savepointSettings.resumeFromLatestCheckpoint()

        executionGraph.getCheckpointCoordinator.restoreSavepoint(
            savepointPath,
            allowNonRestored,
            resumeFromLatestCheckpoint,
            executionGraph.getAllVertices,
            executionGraph.getUserClassLoader
        )
    } catch {
    	...
    }
}
```

- 然后通知client任务提交成功消息，至此job提交成功，但是job还没启动，继续看；

```
jobInfo.notifyClients(
	decorateMessage(JobSubmitSuccess(jobGraph.getJobID)))
```

### 3，**提交Execution给Scheduler进行调度**

#### 3.1，**获取ExecutionGraph中所有vertice，并为其分配slot资源**

- 先判断jobmanager是否是leader,如果是leader,执行scheduleForExecution方法进行调度；否则删除job。

```
if (leaderSessionID.isDefined &&
    leaderElectionService.hasLeadership(leaderSessionID.get)) {
    executionGraph.scheduleForExecution()
} else {
	self ! decorateMessage(RemoveJob(jobId, removeJobFromStateBackend = false))
	log.warn(s"Submitted job $jobId, but not leader. The other leader needs to recover " +
	"this. I am not scheduling the job for execution.")
}
```

- 接着看下scheduleForExecution是如何调度的。
- 如果状态成功从created转变成running,则调用GraphManager开始调度。

```
if (transitionState(JobStatus.CREATED, JobStatus.RUNNING)) {
	graphManager.startScheduling();
}
```

- GraphManager内部其实是使用了graphManagerPlugin的onSchedulingStarted方法；

```
public void startScheduling() {
		LOG.info("Start scheduling execution graph with graph manager plugin: {}",
			graphManagerPlugin.getClass().getName());
		graphManagerPlugin.onSchedulingStarted();
	}
```

**flink实现了三种GraphManagerPlugin：EagerSchedulingPlugin，RunningUnitGraphManagerPlugin，StepwiseSchedulingPlugin。**

- EagerSchedulingPlugin，调度开始后，启动所有顶点；

```
	public void onSchedulingStarted() {
		final List<ExecutionVertexID> verticesToSchedule = new ArrayList<>();
		for (JobVertex vertex : jobGraph.getVerticesSortedTopologicallyFromSources()) {
			for (int i = 0; i < vertex.getParallelism(); i++) {
				verticesToSchedule.add(new ExecutionVertexID(vertex.getID(), i));
			}
		}
		scheduler.scheduleExecutionVertices(verticesToSchedule);
	}
```

- RunningUnitGraphManagerPlugin，根据runningUnit安排作业；

```
public void onSchedulingStarted() {
	runningUnitMap.values().stream()
    	.filter(LogicalJobVertexRunningUnit::allDependReady)
        .forEach(this::addToScheduleQueue);
    checkScheduleNewRunningUnit();
}
```

- StepwiseSchedulingPlugin，首先启动源顶点，并根据其可消耗输入启动下游顶点；

```
public void onSchedulingStarted() {
	final List<ExecutionVertexID> verticesToSchedule = new ArrayList<>();
    for (JobVertex vertex : jobGraph.getVerticesSortedTopologicallyFromSources()) {
    	if (vertex.isInputVertex()) {
    		for (int i = 0; i < vertex.getParallelism(); i++) {
    			verticesToSchedule.add(new ExecutionVertexID(vertex.getID(), i));
    		}
    	}
    }
    scheduleOneByOne(verticesToSchedule);
}
```

上面三种plugin不同是调度vertice的顺序，但是vertice调度方法是一样的，最终都是调用ExecutionGraphVertexScheduler的scheduleExecutionVertices方法；

```
public class ExecutionGraphVertexScheduler implements VertexScheduler {
    public void scheduleExecutionVertices(Collection<ExecutionVertexID> 			         verticesToSchedule) {
        synchronized (executionVerticesToBeScheduled) {
            if (isReconciling) {
                executionVerticesToBeScheduled.add(verticesToSchedule);
                return;
            }
        }
        executionGraph.scheduleVertices(verticesToSchedule);
	}
}
```

- 继续深入scheduleVertices方法，该方法是在调度之前检查vertice健康状态，如果都没问题，则调用schedule(vertices)方法；

```\
public void scheduleVertices(Collection<ExecutionVertexID> verticesToSchedule) {

		try {
			。。。

			final CompletableFuture<Void> schedulingFuture = schedule(vertices);

			if (state == JobStatus.RUNNING && currentGlobalModVersion == globalModVersion) {
				schedulingFutures.put(schedulingFuture, schedulingFuture);
				schedulingFuture.whenCompleteAsync(
						(Void ignored, Throwable throwable) -> {
							schedulingFutures.remove(schedulingFuture);
						},
						futureExecutor);
			} else {
				schedulingFuture.cancel(false);
			}
		} catch (Throwable t) {
			。。。
		}
	}
```

- 深入schedule(vertices)方法，这是真正调度vertices的方法,看看具体做了什么。

  - 为每个vertice准备调度的资源：ScheduledUnit，SlotProfile

  ```
  checkState(state == JobStatus.RUNNING, "job is not running currently");
  		final boolean queued = allowQueuedScheduling;
  		List<SlotRequestId> slotRequestIds = new ArrayList<>(vertices.size());
  		List<ScheduledUnit> scheduledUnits = new ArrayList<>(vertices.size());
  		List<SlotProfile> slotProfiles = new ArrayList<>(vertices.size());
  		List<Execution> scheduledExecutions = new ArrayList<>(vertices.size());
          //为每个vertice准备调度的资源
  		for (ExecutionVertex ev : vertices) {
  			final Execution exec = ev.getCurrentExecutionAttempt();
  			try {
  				Tuple2<ScheduledUnit, SlotProfile> scheduleUnitAndSlotProfile = exec.enterScheduledAndPrepareSchedulingResources();
  				slotRequestIds.add(new SlotRequestId());
  				scheduledUnits.add(scheduleUnitAndSlotProfile.f0);
  				slotProfiles.add(scheduleUnitAndSlotProfile.f1);
  				scheduledExecutions.add(exec);
  			} catch (IllegalExecutionStateException e) {
  				LOG.info("The execution {} may be already scheduled by other thread.", ev.getTaskNameWithSubtaskIndex(), e);
  			}
  		}
  ```

  - 分配slot

  ```
  List<CompletableFuture<LogicalSlot>> allocationFutures =
  				slotProvider.allocateSlots(slotRequestIds, scheduledUnits, queued, slotProfiles, allocationTimeout);
  		List<CompletableFuture<Void>> assignFutures = new ArrayList<>(slotRequestIds.size());
  		for (int i = 0; i < allocationFutures.size(); i++) {
  			final int index = i;
  			allocationFutures.get(i).whenComplete(
  					(ignore, throwable) -> {
  						if (throwable != null) {
  							slotProvider.cancelSlotRequest(
  									slotRequestIds.get(index),
  									scheduledUnits.get(index).getSlotSharingGroupId(),
  									scheduledUnits.get(index).getCoLocationConstraint(),
  									throwable);
  						}
  					}
  			);
  			assignFutures.add(allocationFutures.get(i).thenAccept(
  					(LogicalSlot logicalSlot) -> {
  						if 
  						//
  						(!scheduledExecutions.get(index).tryAssignResource(logicalSlot)) {
  							// release the slot
  							Exception e = new FlinkException("Could not assign logical slot to execution " + scheduledExecutions.get(index) + '.');
  							logicalSlot.releaseSlot(e);
  							throw new CompletionException(e);
  						}
  					})
  			);
  		}
  ```

  - 所有的slot分配完成才算完成，有一个失败便算失败。所有slot分配成功之后，异步执行所有Exceution的deploy方法。

  ```
  CompletableFuture<Void> currentSchedulingFuture = allAssignFutures
      .异常处理
      .handleAsync(
      (Collection<Void> ignored, Throwable throwable) -> {
      	if (throwable != null) {
      		throw new CompletionException(throwable);
      	} else {
      		boolean hasFailure = false;
      		for (int i = 0; i <  scheduledExecutions.size(); i++) {
      			try {
      				scheduledExecutions.get(i).deploy();
      			} catch (Exception e) {
      				hasFailure = true;
      				scheduledExecutions.get(i).markFailed(e);
      		}
      	}
          if (hasFailure) {
              throw new CompletionException(
              new FlinkException("Fail to deploy some executions."));
          }
      }
      return null;
  }, futureExecutor);
  ```

#### 3.2，**通知TaskManager，将每个vertice部署在分配好的资源中**

下面深入deploy方法，deploy负责将Execution部署到先前分配好的资源上，提交task到taskManagerGateway，然后由taskManagerGateway转发给Taskmanager。TaskManager如何处理SubmitTask消息之后分析。

```
public void deploy() throws JobException {
		...一系列检查保证slot可用
		executor.execute(
			() -> {
				try {
					final TaskDeploymentDescriptor deployment = vertex.createDeploymentDescriptor(
							attemptId,
							slot,
							taskRestore,
							attemptNumber);

					// null taskRestore to let it be GC'ed
					taskRestore = null;

					final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();
					//提交task到taskManagerGateway，然后由taskManagerGateway转发给Taskmanager
					final CompletableFuture<Acknowledge> submitResultFuture = taskManagerGateway.submitTask(deployment, rpcTimeout);

					...
				} catch (Throwable t) {
					markFailed(t);
				}
			}
		);
	}
```