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