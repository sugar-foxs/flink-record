## JobManager
- jobManager 负责接收flink jobs，任务调度，收集job状态，和管理taskManager
  继承了FlinkActor，它主要接收以下信息：
  - RegisterTaskManager: 它由想要注册到JobManager的TaskManager发送。注册成功会通过AcknowledgeRegistration消息进行Ack。
  - SubmitJob: 由提交作业到系统的Client发送。提交的信息是JobGraph形式的作业描述信息。
  - CancelJob: 请求取消指定id的作业。成功会返回CancellationSuccess，否则返回CancellationFailure。
  - UpdateTaskExecutionState: 由TaskManager发送，用来更新执行节点(ExecutionVertex)的状态。成功则返回true，否则返回false。
  - RequestNextInputSplit: TaskManager上的Task请求下一个输入split，成功则返回NextInputSplit，否则返回null。
  - JobStatusChanged： 它意味着作业的状态(RUNNING, CANCELING, FINISHED,等)发生变化。这个消息由ExecutionGraph发送。

## JobManager启动
- JobManager里有个main方法,通过脚本启动，主要做的工作如下：
- 首先做一些检查，加载配置工作
- 启动jobManager
        
        //加载安全配置
		SecurityUtils.install(new SecurityConfiguration(configuration))
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

-追踪至 runJobManager方法，下面着重看下这个方法：

	def runJobManager(
      configuration: Configuration,
      executionMode: JobManagerMode,
      listeningAddress: String,
      listeningPort: Int)
    : Unit = {

    val numberProcessors = Hardware.getNumberCPUCores()

    val futureExecutor = Executors.newScheduledThreadPool(
      numberProcessors,
      new ExecutorThreadFactory("jobmanager-future"))

    val ioExecutor = Executors.newFixedThreadPool(
      numberProcessors,
      new ExecutorThreadFactory("jobmanager-io"))

    val timeout = AkkaUtils.getTimeout(configuration)

    // 首先启动ActorSystem，因为如果端口0之前被选中了，startActorSystem方法决定了使用哪个端口号，并进行相应的更新。
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

    try {
      highAvailabilityServices.close()
    } catch {
      case t: Throwable =>
        LOG.warn("Could not properly stop the high availability services.", t)
    }

    try {
      metricRegistry.shutdown().get()
    } catch {
      case t: Throwable =>
        LOG.warn("Could not properly shut down the metric registry.", t)
    }

    ExecutorUtils.gracefulShutdown(
      timeout.toMillis,
      TimeUnit.MILLISECONDS,
      futureExecutor,
      ioExecutor)
    }

### jobManager处理消息
- handleMessage方法，包含以下类型消息

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