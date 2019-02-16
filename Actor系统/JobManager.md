## JobManager

- jobManager 负责接收flink jobs，任务调度，收集job状态，和管理taskManager

  继承自FlinkActor，它主要接收以下信息：
  - RegisterTaskManager: 它由想要注册到JobManager的TaskManager发送。注册成功会通过AcknowledgeRegistration消息进行Ack。
  - SubmitJob: 由提交作业到系统的Client发送。提交的信息是JobGraph形式的作业描述信息，所有flink作业都会抽象成JobGraph交给JobManager处理。
  - CancelJob: 请求取消指定id的作业。成功会返回CancellationSuccess，否则返回CancellationFailure。
  - UpdateTaskExecutionState: 由TaskManager发送，用来更新执行节点(ExecutionVertex)的状态。成功则返回true，否则返回false。
  - RequestNextInputSplit: TaskManager上的Task请求下一个输入split，成功则返回NextInputSplit，否则返回null。
  - JobStatusChanged： 它意味着作业的状态(RUNNING, CANCELING, FINISHED,等)发生变化。这个消息由ExecutionGraph通知。

## JobManager启动

- JobManager里有个main方法,通过脚本启动，主要做的工作如下：

- 首先做一些检查，加载日志环境和配置

- 启动jobManager 

  ```
  try {
    SecurityUtils.getInstalledContext.runSecured(new Callable[Unit] {
      override def call(): Unit = {
        runJobManager(
          configuration,
          executionMode,
          externalHostName,
          portRange)
      }
    })
  } catch {
    。。。
  }
  ```

- 深入runJobManager方法，下面着重看下这个方法：

```
def runJobManager(
  configuration: Configuration,
  executionMode: JobManagerMode,
  listeningAddress: String,
  listeningPort: Int)
: Unit = {

。。。

// 首先启动JobManager ActorSystem，因为如果端口0之前被选中了，startActorSystem方法决定了使用哪个端口号，并进行相应的更新。
val jobManagerSystem = startActorSystem(
  configuration,
  listeningAddress,
  listeningPort)

//创建HA服务
val highAvailabilityServices = HighAvailabilityServicesUtils.createHighAvailabilityServices(
  configuration,
  ioExecutor,
  AddressResolution.NO_ADDRESS_RESOLUTION)

val metricRegistry = new MetricRegistryImpl(
  MetricRegistryConfiguration.fromConfiguration(configuration))
//启动metrics查询服务
metricRegistry.startQueryService(jobManagerSystem, null)
//启动jobManager所有组件
val (_, _, webMonitorOption, _) = try {
  startJobManagerActors(
    jobManagerSystem,
    configuration,
    executionMode,
    listeningAddress,
    futureExecutor,
    ioExecutor,
    highAvailabilityServices,
    metricRegistry,
    classOf[JobManager],
    classOf[MemoryArchivist],
    Option(classOf[StandaloneResourceManager])
  )
} catch {
  ...
}

// 阻塞 直至系统被shutdown
jobManagerSystem.awaitTermination()
//下面关闭所有服务
webMonitorOption.foreach{
  webMonitor =>
    try {
      webMonitor.stop()
    } catch {
      case t: Throwable =>
        LOG.warn("Could not properly stop the web monitor.", t)
    }
}
...
```
- 看下startJobManagerActors方法，启动了哪些组件
```
    。。。
    try {
      // 启动Jobmanager actor
      val (jobManager, archive) = startJobManagerActors(
        configuration,
        jobManagerSystem,
        futureExecutor,
        ioExecutor,
        highAvailabilityServices,
        metricRegistry,
        webMonitor.map(_.getRestAddress),
        jobManagerClass,
        archiveClass)
      
      // 启动JobManager process reaper,在Jobmanager失败时来结束jobmanager jvm进程
      jobManagerSystem.actorOf(
        Props(
          classOf[ProcessReaper],
          jobManager,
          LOG.logger,
          RUNTIME_FAILURE_RETURN_CODE),
        "JobManager_Process_Reaper")

      // 如果是本地模式，启动一个本地taskmanager
      if (executionMode == JobManagerMode.LOCAL) {
        val resourceId = ResourceID.generate()
        val taskManagerActor = TaskManager.startTaskManagerComponentsAndActor(
          configuration,
          resourceId,
          jobManagerSystem,
          highAvailabilityServices,
          metricRegistry,
          externalHostname,
          Some(TaskExecutor.TASK_MANAGER_NAME),
          localTaskManagerCommunication = true,
          classOf[TaskManager])

        jobManagerSystem.actorOf(
          Props(
            classOf[ProcessReaper],
            taskManagerActor,
            LOG.logger,
            RUNTIME_FAILURE_RETURN_CODE),
          "TaskManager_Process_Reaper")
      }

      。。。省略了webmonitor

      val resourceManager =
        resourceManagerClass match {
          case Some(rmClass) =>
            //启动Resource manager actor
            Option(
              FlinkResourceManager.startResourceManagerActors(
                configuration,
                jobManagerSystem,
                highAvailabilityServices.getJobManagerLeaderRetriever(
                  HighAvailabilityServices.DEFAULT_JOB_ID),
                rmClass))
          case None =>
            LOG.info("Resource Manager class not provided. No resource manager will be started.")
            None
        }

      (jobManager, archive, webMonitor, resourceManager)
    }
    catch {
      ...
    }
```
综上，启动了jobmanager和监控jobmanager的reaper，本地模式下启动taskmanager和监控taskmanager的reaper，resourcemanager。

- 继续深入到startJobManagerActors方法
```
def startJobManagerActors(
      configuration: Configuration,
      actorSystem: ActorSystem,
      futureExecutor: ScheduledExecutorService,
      ioExecutor: Executor,
      highAvailabilityServices: HighAvailabilityServices,
      metricRegistry: FlinkMetricRegistry,
      optRestAddress: Option[String],
      jobManagerActorName: Option[String],
      archiveActorName: Option[String],
      jobManagerClass: Class[_ <: JobManager],
      archiveClass: Class[_ <: MemoryArchivist])
    : (ActorRef, ActorRef) = {

    val (instanceManager,
    scheduler,
    blobServer,
    libraryCacheManager,
    restartStrategy,
    timeout,
    archiveCount,
    archivePath,
    jobRecoveryTimeout,
    jobManagerMetricGroup) = createJobManagerComponents(
      configuration,
      futureExecutor,
      ioExecutor,
      highAvailabilityServices.createBlobStore(),
      metricRegistry)

    val archiveProps = getArchiveProps(archiveClass, archiveCount, archivePath)

    // start the archiver with the given name, or without (avoid name conflicts)
    val archive: ActorRef = archiveActorName match {
      case Some(actorName) => actorSystem.actorOf(archiveProps, actorName)
      case None => actorSystem.actorOf(archiveProps)
    }

    val jobManagerProps = getJobManagerProps(
      jobManagerClass,
      configuration,
      futureExecutor,
      ioExecutor,
      instanceManager,
      scheduler,
      blobServer,
      libraryCacheManager,
      archive,
      restartStrategy,
      timeout,
      highAvailabilityServices.getJobManagerLeaderElectionService(
        HighAvailabilityServices.DEFAULT_JOB_ID),
      highAvailabilityServices.getSubmittedJobGraphStore(),
      highAvailabilityServices.getCheckpointRecoveryFactory(),
      jobRecoveryTimeout,
      jobManagerMetricGroup,
      optRestAddress)

    val jobManager: ActorRef = jobManagerActorName match {
      case Some(actorName) => actorSystem.actorOf(jobManagerProps, actorName)
      case None => actorSystem.actorOf(jobManagerProps)
    }

    (jobManager, archive)
  }
```
createJobManagerComponents方法创建了jobmanager组件，包括了存储、备份等策略的组件实现，还包括scheduler、submittedJobGraphs，分别负责job的调度和作业的提交。最后返回archive和jobmanager。


### jobManager处理消息
jobmanager创建成功之后，如何处理消息呢？JobManager继承自FlinkActor，重写了handleMessage方法，用来处理消息。
- handleMessage方法，包含以下类型消息，具体每个消息做什么工作以后添加。

  ```
  //通知哪个jobManager作为leader
  case GrantLeadership(newLeaderSessionID) 
  //撤销leader
  case RevokeLeadership
    //注册资源管理器
  case msg: RegisterResourceManager 
    //重新连接资源管理器
  case msg: ReconnectResourceManager 
  //注册taskManager
  case msg @ RegisterTaskManager(
        resourceId,
        connectionInfo,
        hardwareInformation,
        numberOfSlots)
  // 由资源管理器发出，某个资源不可用
  case msg: ResourceRemoved 
  // 请求已经注册的taskManager的数量
  case RequestNumberRegisteredTaskManager 
  // 请求slot总量
  case RequestTotalNumberOfSlots
  // 由client发出的提交job消息
  case SubmitJob(jobGraph, listeningBehaviour) 
  //注册job client
  case RegisterJobClient(jobID, listeningBehaviour)
  // 恢复已经提交的job
  case RecoverSubmittedJob(submittedJobGraph) 
  // 恢复job
  case RecoverJob(jobId)
  // 恢复所有jobs
  case RecoverAllJobs 
  // 取消job
  case CancelJob(jobID) 
  // 取消job并设置保存点
  case CancelJobWithSavepoint(jobId, savepointDirectory) 
  // 停止job
  case StopJob(jobID) 
  // 更新task执行状态
  case UpdateTaskExecutionState(taskExecutionState)
  // 
  case RequestNextInputSplit(jobID, vertexID, executionAttempt) 
  // checkpoint相关消息
  case checkpointMessage : AbstractCheckpointMessage 
  // 
  case kvStateMsg : KvStateMessage
  // 触发回到最新savePoint恢复执行
  case TriggerSavepoint(jobId, savepointDirectory) 
  // 丢弃某个savePoint
  case DisposeSavepoint(savepointPath) 
  // job状态更新
  case msg @ JobStatusChanged(jobID, newJobStatus, timeStamp, error) 
  // 调度或更新消费者
  case ScheduleOrUpdateConsumers(jobId, partitionId) 
  
  case RequestPartitionProducerState(jobId, intermediateDataSetId, resultPartitionId) 
  // 请求job状态
  case RequestJobStatus(jobID) 
  // 请求所有运行的job的executionGraph
  case RequestRunningJobs
  // 请求所有job状态
  case RequestRunningJobsStatus 
  //请求某个job信息,包含jobID和executionGraph
  case RequestJob(jobID)
  // 请求classloading配置
  case RequestClassloadingProps(jobID)
  // 获得blobServer port
  case RequestBlobManagerPort 
  // 
  case RequestArchive 
  // 获得所有注册的taskManager
  case RequestRegisteredTaskManagers 
  // 
  case RequestTaskManagerInstance(resourceId) 
  // 心跳
  case Heartbeat(instanceID, accumulators) 
  // 
  case message: AccumulatorMessage 
  // 包含RequestJobsOverview,RequestJobsWithIDsOverview,RequestStatusOverview,
    //RequestJobDetails
  case message: InfoMessage 
  // 请求堆栈信息
  case RequestStackTrace(instanceID) 
  // 关闭taskManager
  case Terminated(taskManagerActorRef) 
  // 请求jobManager状态
  case RequestJobManagerStatus 
  // 删除job
  case RemoveJob(jobID, clearPersistedJob) 
  // 删除缓存的job
  case RemoveCachedJob(jobID) 
  // taskManager要求断开连接
  case Disconnect(instanceId, cause) 
  // 停止所有的taskManager
  case msg: StopCluster 
  // 请求leader id
  case RequestLeaderSessionID 
  // 获取restAddress
  case RequestRestAddress 
  ```

## jobmanager处理SubmitJob消息

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

- 注册Job状态变化的事件回调;

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

- 如何生成executionGraph在另一个文章中分析，获取executionGraph之后，如何提交job，继续看；
- 如果是恢复的job，从最新的checkpoint中恢复。

```
if (isRecovery) {
    // this is a recovery of a master failure (this master takes over)
    executionGraph.restoreLatestCheckpointedState(false, false)
}
```

- 或者获取savepoint配置，如果配置了savepoint，便从savepoint中恢复

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

- 然后通知client jobsubmit成功消息；

```
jobInfo.notifyClients(
	decorateMessage(JobSubmitSuccess(jobGraph.getJobID)))
```

- 如果这个jobManager是leader,执行scheduleForExecution方法进行调度；否则删除job。

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
- **状态从created转变成running,然后调用GraphManager开始调度。**

```
if (transitionState(JobStatus.CREATED, JobStatus.RUNNING)) {
	graphManager.startScheduling();
}
```

- 其实是使用了graphManagerPlugin的onSchedulingStarted方法；

```
public void startScheduling() {
		LOG.info("Start scheduling execution graph with graph manager plugin: {}",
			graphManagerPlugin.getClass().getName());
		graphManagerPlugin.onSchedulingStarted();
	}
```

### GraphManagerPlugin

**flink实现了三种GraphManagerPlugin：EagerSchedulingPlugin，RunningUnitGraphManagerPlugin，StepwiseSchedulingPlugin。**

#### EagerSchedulingPlugin

- 调度开始后，启动所有顶点。这个比较粗暴，生成所有的顶点并调度。

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

#### RunningUnitGraphManagerPlugin

- 根据runningUnit安排作业

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



#### StepwiseSchedulingPlugin

- 首先启动源顶点，并根据其可消耗输入启动下游顶点

```
public void onSchedulingStarted() {
		runningUnitMap.values().stream()
				.filter(LogicalJobVertexRunningUnit::allDependReady)
				.forEach(this::addToScheduleQueue);
		checkScheduleNewRunningUnit();
	}
```

上面三种不同是调度vertice的顺序，但是vertice调度方法是一样的，都是调用ExecutionGraphVertexScheduler的scheduleExecutionVertices方法；

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
			long currentGlobalModVersion = globalModVersion;

			if (verticesToSchedule == null || verticesToSchedule.isEmpty()) {
				return;
			}

			int alreadyScheduledCount = 0;
			int unhealthyCount = 0;
			final List<ExecutionVertex> vertices = new ArrayList<>(verticesToSchedule.size());
			for (ExecutionVertexID executionVertexID : verticesToSchedule) {
				ExecutionVertex ev = getJobVertex(executionVertexID.getJobVertexID()).getTaskVertices()[executionVertexID.getSubTaskIndex()];
				if (ev.getExecutionState() == ExecutionState.CREATED) {
					vertices.add(ev);
				} else if (ev.getExecutionState() == ExecutionState.SCHEDULED ||
						ev.getExecutionState() == ExecutionState.DEPLOYING ||
						ev.getExecutionState() == ExecutionState.RUNNING ||
						ev.getExecutionState() == ExecutionState.FINISHED) {

					alreadyScheduledCount++;
				} else {
					unhealthyCount++;
				}
			}
			if (unhealthyCount >  0) {
				throw new IllegalStateException("Not all submitted vertices can be scheduled. " +
//						alreadyScheduledCount + " vertices are already scheduled and " +
						unhealthyCount + " vertices are in unhealthy state. " +
						"Please check the schedule logic in " + graphManager.getClass().getCanonicalName());
			}

			LOG.info("Schedule {} vertices: {}, already scheduled {}", vertices.size(), vertices, alreadyScheduledCount);

			if (vertices.isEmpty()) {
				return;
			}

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

```
private CompletableFuture<Void> schedule(Collection<ExecutionVertex> vertices) {

		checkState(state == JobStatus.RUNNING, "job is not running currently");
		final boolean queued = allowQueuedScheduling;
		List<SlotRequestId> slotRequestIds = new ArrayList<>(vertices.size());
		List<ScheduledUnit> scheduledUnits = new ArrayList<>(vertices.size());
		List<SlotProfile> slotProfiles = new ArrayList<>(vertices.size());
		List<Execution> scheduledExecutions = new ArrayList<>(vertices.size());

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

		if (slotRequestIds.isEmpty()) {
			return CompletableFuture.completedFuture(null);
		}
        //分配slot
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
		// 所有的slot分配完成才算完成，有一个失败便算失败。
		final ConjunctFuture<Collection<Void>> allAssignFutures = FutureUtils.combineAll(assignFutures);
		。。。省略异常处理
		return currentSchedulingFuture;
	}
```

所以上述方法主要任务是为vertice分配slot资源。

