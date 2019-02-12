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

### ActorTaskManagerGateway(与Taskmanager直接联系)

	public CompletableFuture<Acknowledge> submitTask(TaskDeploymentDescriptor tdd, Time timeout) {
			Preconditions.checkNotNull(tdd);
			Preconditions.checkNotNull(timeout);
	
			scala.concurrent.Future<Acknowledge> submitResult = actorGateway.ask(
	            //这个是Taskmanager接收到的SubmitTask				
				new TaskMessages.SubmitTask(tdd),
				new FiniteDuration(timeout.getSize(), timeout.getUnit()))
				.mapTo(ClassTag$.MODULE$.<Acknowledge>apply(Acknowledge.class));
	
			return FutureUtils.toJava(submitResult);
	}

等下看下task如何调度的
### executionGraph生成过程
由ExecutionGraphBuilder.buildGraph()生成，