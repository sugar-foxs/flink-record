## JobManager处理的消息

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

