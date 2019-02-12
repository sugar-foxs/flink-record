## TaskManager
- TaskManager负责执行Flink job的各个任务，其有以下几个阶段：
	* 等待被注册： 周期的发送RegisterTaskManager消息给jobManager，直到注册成功，jobManager会回复AcknowledgeRegistration消息，然后taskManager初始化所有组件。
	* 接收处理消息：如，SubmitTask, CancelTask, FailTask

## TaskManager 启动
- 首先HA服务，获取网络端口，重点在runManager方法

		def runTaskManager(
		      taskManagerHostname: String,
		      resourceID: ResourceID,
		      actorSystemPort: Int,
		      configuration: Configuration,
		      highAvailabilityServices: HighAvailabilityServices,
		      taskManagerClass: Class[_ <: TaskManager])
		    : Unit = {
		
		    // 首先创建TaskManager actor system，并绑定地址
		    val taskManagerSystem = BootstrapTools.startActorSystem(
		      configuration,
		      taskManagerHostname,
		      actorSystemPort,
		      LOG.logger)
		
		    val metricRegistry = new MetricRegistryImpl(
		      MetricRegistryConfiguration.fromConfiguration(configuration))
		
		    metricRegistry.startQueryService(taskManagerSystem, resourceID)
		
		    // 开启所有taskManager组件和actor
		    try {
		      LOG.info("Starting TaskManager actor")
		      val taskManager = startTaskManagerComponentsAndActor(
		        configuration,
		        resourceID,
		        taskManagerSystem,
		        highAvailabilityServices,
		        metricRegistry,
		        taskManagerHostname,
		        Some(TaskExecutor.TASK_MANAGER_NAME),
		        localTaskManagerCommunication = false,
		        taskManagerClass)
		
		      // 开启监听jobManager的进程，如果taskManager actor die，可以kill JVM进程
		      LOG.debug("Starting TaskManager process reaper")
		      taskManagerSystem.actorOf(
		        Props(classOf[ProcessReaper], taskManager, LOG.logger, RUNTIME_FAILURE_RETURN_CODE),
		        "TaskManager_Process_Reaper")
		
		      if (configuration.getBoolean(TaskManagerOptions.EXIT_ON_FATAL_AKKA_ERROR)) {
		        val quarantineHandler = new DefaultQuarantineHandler(
		          Time.milliseconds(AkkaUtils.getTimeout(configuration).toMillis),
		          RUNTIME_FAILURE_RETURN_CODE,
		          LOG.logger)
		
		        LOG.debug("Starting TaskManager quarantine monitor")
		        taskManagerSystem.actorOf(
		          Props(classOf[QuarantineMonitor], quarantineHandler, LOG.logger)
		        )
		      }
		
		      // 日志相关
		      if (LOG.isInfoEnabled && configuration.getBoolean(
		        TaskManagerOptions.DEBUG_MEMORY_LOG))
		      {
		        LOG.info("Starting periodic memory usage logger")
		
		        val interval = configuration.getLong(
		          TaskManagerOptions.DEBUG_MEMORY_USAGE_LOG_INTERVAL_MS)
		
		        val logger = new MemoryLogger(LOG.logger, interval, taskManagerSystem)
		        logger.start()
		      }
		
		      // 阻塞，直至系统关闭
		      taskManagerSystem.awaitTermination()
		    } catch {
		      。。。
		    }
		
		    // 关闭metrics查询服务
		    try {
		      metricRegistry.shutdown().get()
		    } catch {
		     。。。
		    }
		  }

	
			

			//等待future超时时间
			prected val resources = HardwareDescription.extractFromSystem(memoryManager.getMemorySize())
			otected val askTimeout = new Timeout(config.getTimeout().getSize, config.getTimeout().getUnit())
			
			//TaskManager执行需要的资源，包含numberOfCPUCores, sizeOfPhysicalMemory, sizeOfJvmHeap, managedMemory
			prot
			//存储TaskManager当前执行的所有Task的map
			protected val runningTasks = new java.util.HashMap[ExecutionAttemptID, Task]()
			
			//用来在task之间广播变量
			protected val bcVarManager = new BroadcastVariableManager()
			
			//用来缓存分布式文件
			protected val fileCache = new FileCache(config.getTmpDirectories())
			
			protected val leaderRetrievalService: LeaderRetrievalService = highAvailabilityServices.
			getJobManagerLeaderRetriever(
			  HighAvailabilityServices.DEFAULT_JOB_ID)
			
			private var taskManagerMetricGroup : TaskManagerMetricGroup = _
			
			//存储了在taskManager被jobmanager注册之后需要通知的的ActorRef(akka)
			private val waitForRegistration = scala.collection.mutable.Set[ActorRef]()
			//用来处理上传TaskManager log的服务
			private var blobService: Option[BlobService] = None
			private var libraryCacheManager: Option[LibraryCacheManager] = None
			
			/* The current leading JobManager Actor associated with */
			protected var currentJobManager: Option[ActorRef] = None
			/* The current leading JobManager URL */
			private var jobManagerAkkaURL: Option[String] = None
			
			private var instanceID: InstanceID = null
			
			private var heartbeatScheduler: Option[Cancellable] = None
			
			var leaderSessionID: Option[UUID] = None
			
			private var scheduledTaskManagerRegistration: Option[Cancellable] = None
			private var currentRegistrationRun: UUID = UUID.randomUUID()
			
			private var connectionUtils: Option[(
			CheckpointResponder,
			PartitionProducerStateChecker,
			ResultPartitionConsumableNotifier,
			TaskManagerActions)] = None

 

### TaskManager处理接收到的messages
    override def handleMessage: Receive = {
    // 处理task messages，任务调度和任务状态更新相关
    case message: TaskMessage => handleTaskMessage(message)

    // 处理checkpointing message
    case message: AbstractCheckpointMessage => handleCheckpointingMessage(message)

    // 处理更换JobManager Leader消息
    case JobManagerLeaderAddress(address, newLeaderSessionID) =>
      handleJobManagerLeaderAddress(address, newLeaderSessionID)

    // 处理注册JobManager相关消息，例：注册被拒绝，已经注册...
    case message: RegistrationMessage => handleRegistrationMessage(message)

    // 处理任务采样消息
    case message: StackTraceSampleMessages => handleStackTraceSampleMessage(message)

    // 发送心跳信息给jobManager
    case SendHeartbeat => sendHeartbeatToJobManager()

    // 返回TaskManager堆栈跟踪
    case SendStackTrace => sendStackTrace(sender())

    // 在注册到jobManager之后通知消息发送者
    case NotifyWhenRegisteredAtJobManager =>
      if (isConnected) {
        sender ! decorateMessage(RegisteredAtJobManager(leaderSessionID.orNull))
      } else {
        waitForRegistration += sender
      }

    // 这个消息表明actor已经关闭
    case Terminated(actor: ActorRef) =>
      if (isConnected && actor == currentJobManager.orNull) {
          handleJobManagerDisconnect("JobManager is no longer reachable")
          triggerTaskManagerRegistration()
      } else {
        log.warn(s"Received unrecognized disconnect message " +
            s"from ${if (actor == null) null else actor.path}.")
      }

    //这个消息表明jobManager需要断开连接
    case Disconnect(instanceIdToDisconnect, cause) =>
      if (instanceIdToDisconnect.equals(instanceID)) {
        handleJobManagerDisconnect(s"JobManager requested disconnect: ${cause.getMessage()}")
        triggerTaskManagerRegistration()
      } else {
        log.debug(s"Received disconnect message for wrong instance id ${instanceIdToDisconnect}.")
      }

    //停止集群消息
    case msg: StopCluster =>
      log.info(s"Stopping TaskManager with final application status ${msg.finalStatus()} " +
        s"and diagnostics: ${msg.message()}")
      shutdown()

    //发生重大错误，要求kill TaskManager actor
    case FatalError(message, cause) =>
      killTaskManagerFatal(message, cause)

    // 请求TaskManager 日志
    case RequestTaskManagerLog(requestType : LogTypeRequest) =>
      blobService match {
        case Some(_) =>
          handleRequestTaskManagerLog(sender(), requestType, currentJobManager.get)
        case None =>
          sender() ! akka.actor.Status.Failure(new IOException("BlobService not " +
            "available. Cannot upload TaskManager logs."))
      }

    // 请求广播变量数目
    case RequestBroadcastVariablesWithReferences =>
      sender ! decorateMessage(
        ResponseBroadcastVariablesWithReferences(
          bcVarManager.getNumberOfVariablesWithReferences)
      )

    // 请求存活连接数
    case RequestNumActiveConnections =>
      val numActive = if (!network.isShutdown) {
        network.getConnectionManager.getNumberOfActiveConnections
      } else {
        0
      }

      sender ! decorateMessage(ResponseNumActiveConnections(numActive))
    }

### 处理与task相关的message
    private def handleTaskMessage(message: TaskMessage): Unit = {
    // 判断是否连接jobManager
    if (!isConnected) {
      log.debug(s"Dropping message $message because the TaskManager is currently " +
        "not connected to a JobManager.")
    } else {
      // we order the messages by frequency, to make sure the code paths for matching
      // are as short as possible
      message match {

		// tell the task about the availability of a new input partition
		case UpdateTaskSinglePartitionInfo(executionID, resultID, partitionInfo) =>
		  updateTaskInputPartitions(
		    executionID,
		    Collections.singletonList(new PartitionInfo(resultID, partitionInfo)))
		
		// tell the task about the availability of some new input partitions
		case UpdateTaskMultiplePartitionInfos(executionID, partitionInfos) =>
		  updateTaskInputPartitions(executionID, partitionInfos)
		
		// discards intermediate result partitions of a task execution on this TaskManager
		case FailIntermediateResultPartitions(executionID) =>
		  log.info("Discarding the results produced by task execution " + executionID)
		  try {
		    network.getResultPartitionManager.releasePartitionsProducedBy(executionID)
		  } catch {
		    case t: Throwable => killTaskManagerFatal(
		    "Fatal leak: Unable to release intermediate result partition data", t)
		  }
		
		// notifies the TaskManager that the state of a task has changed.
		// the TaskManager informs the JobManager and cleans up in case the transition
		// was into a terminal state, or in case the JobManager cannot be informed of the
		// state transition

      // 更新task执行状态
      case updateMsg @ UpdateTaskExecutionState(taskExecutionState: TaskExecutionState) =>

          // we receive these from our tasks and forward them to the JobManager
          currentJobManager foreach {
            jobManager => {
            val futureResponse = (jobManager ? decorateMessage(updateMsg))(askTimeout)

              val executionID = taskExecutionState.getID

              futureResponse.mapTo[Boolean].onComplete {
                // IMPORTANT: In the future callback, we cannot directly modify state
                //            but only send messages to the TaskManager to do those changes
                case scala.util.Success(result) =>
                  if (!result) {
                  self ! decorateMessage(
                    FailTask(
                      executionID,
                      new Exception("Task has been cancelled on the JobManager."))
                    )
                  }

                case scala.util.Failure(t) =>
                self ! decorateMessage(
                  FailTask(
                    executionID,
                    new Exception(
                      "Failed to send ExecutionStateChange notification to JobManager", t))
                )
              }(context.dispatcher)
            }
          }

        // 从taskManager删除task
        case TaskInFinalState(executionID) =>
          unregisterTaskAndNotifyFinalState(executionID)

        // 提交任务，tdd包含所有开启task需要的信息
        case SubmitTask(tdd) =>
          submitTask(tdd)

        // 标记因外部原因失败的任务
        case FailTask(executionID, cause) =>
          val task = runningTasks.get(executionID)
          if (task != null) {
            task.failExternally(cause)
          } else {
            log.debug(s"Cannot find task to fail for execution $executionID)")
          }

        // 停止task
        case StopTask(executionID) =>
          val task = runningTasks.get(executionID)
          if (task != null) {
            try {
              task.stopExecution()
              sender ! decorateMessage(Acknowledge.get())
            } catch {
              case t: Throwable =>
                sender ! decorateMessage(Status.Failure(t))
            }
          } else {
            log.debug(s"Cannot find task to stop for execution ${executionID})")
            sender ! decorateMessage(Acknowledge.get())
          }
 
        // 取消task
        case CancelTask(executionID) =>
          val task = runningTasks.get(executionID)
          if (task != null) {
            task.cancelExecution()
            sender ! decorateMessage(Acknowledge.get())
          } else {
            log.debug(s"Cannot find task to cancel for execution $executionID)")
            sender ! decorateMessage(Acknowledge.get())
          }
      }
      }
  }

### 处理checkpoint相关消息
    private def handleCheckpointingMessage(actorMessage:    AbstractCheckpointMessage): Unit = {

    actorMessage match {
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

      case message: NotifyCheckpointComplete =>
        val taskExecutionId = message.getTaskExecutionId
        val checkpointId = message.getCheckpointId
        val timestamp = message.getTimestamp

        log.debug(s"Receiver ConfirmCheckpoint $checkpointId@$timestamp for $taskExecutionId.")

        val task = runningTasks.get(taskExecutionId)
        if (task != null) {
          task.notifyCheckpointComplete(checkpointId)
        } else {
          log.debug(
            s"TaskManager received a checkpoint confirmation for unknown task $taskExecutionId.")
        }

      // unknown checkpoint message
      case _ => unhandled(actorMessage)
    }
  }

### 处理注册相关消息
    private def handleRegistrationMessage(message: RegistrationMessage): Unit = {
    message match {
      case TriggerTaskManagerRegistration(
        jobManagerURL,
        timeout,
        deadline,
        attempt,
        registrationRun) =>

        if (registrationRun.equals(this.currentRegistrationRun)) {
          if (isConnected) {
            // this may be the case, if we queue another attempt and
            // in the meantime, the registration is acknowledged
            log.debug(
              "TaskManager was triggered to register at JobManager, but is already registered")
          } else if (deadline.exists(_.isOverdue())) {
            // we failed to register in time. that means we should quit
            log.error("Failed to register at the JobManager withing the defined maximum " +
                        "connect time. Shutting down ...")

            // terminate ourselves (hasta la vista)
            self ! decorateMessage(PoisonPill)
          } else {
            if (!jobManagerAkkaURL.equals(Option(jobManagerURL))) {
              throw new Exception("Invalid internal state: Trying to register at JobManager " +
                                    s"$jobManagerURL even though the current JobManagerAkkaURL " +
                                    s"is set to ${jobManagerAkkaURL.getOrElse("")}")
            }

            log.info(s"Trying to register at JobManager $jobManagerURL " +
                       s"(attempt $attempt, timeout: $timeout)")

            val jobManager = context.actorSelection(jobManagerURL)

            jobManager ! decorateMessage(
              RegisterTaskManager(
                resourceID,
                location,
                resources,
                numberOfSlots)
            )

            // the next timeout computes via exponential backoff with cap
            val nextTimeout = (timeout * 2).min(new FiniteDuration(
              config.getMaxRegistrationPause().toMilliseconds,
              TimeUnit.MILLISECONDS))

            // schedule (with our timeout s delay) a check triggers a new registration
            // attempt, if we are not registered by then
            scheduledTaskManagerRegistration = Option(context.system.scheduler.scheduleOnce(
              timeout,
              self,
              decorateMessage(TriggerTaskManagerRegistration(
                jobManagerURL,
                nextTimeout,
                deadline,
                attempt + 1,
                registrationRun)
              ))(context.dispatcher))
          }
        } else {
          log.info(s"Discarding registration run with ID $registrationRun")
        }

      // successful registration. associate with the JobManager
      // we disambiguate duplicate or erroneous messages, to simplify debugging
      case AcknowledgeRegistration(id, blobPort) =>
        val jobManager = sender()

        if (isConnected) {
          if (jobManager == currentJobManager.orNull) {
            log.debug("Ignoring duplicate registration acknowledgement.")
          } else {
            log.warn(s"Ignoring 'AcknowledgeRegistration' message from ${jobManager.path} , " +
              s"because the TaskManager is already registered at ${currentJobManager.orNull}")
          }
        } else {
          // not yet connected, so let's associate with that JobManager
          try {
            associateWithJobManager(jobManager, id, blobPort)
          } catch {
            case t: Throwable =>
              killTaskManagerFatal(
                "Unable to start TaskManager components and associate with the JobManager", t)
          }
        }

      // we are already registered at that specific JobManager - duplicate answer, rare cases
      case AlreadyRegistered(id, blobPort) =>
        val jobManager = sender()

        if (isConnected) {
          if (jobManager == currentJobManager.orNull) {
            log.debug("Ignoring duplicate registration acknowledgement.")
          } else {
            log.warn(s"Received 'AlreadyRegistered' message from " +
              s"JobManager ${jobManager.path}, even through TaskManager is currently " +
              s"registered at ${currentJobManager.orNull}")
          }
        } else {
          // not connected, yet, so let's associate
          log.info("Received 'AlreadyRegistered' message before 'AcknowledgeRegistration'")

          try {
            associateWithJobManager(jobManager, id, blobPort)
          } catch {
            case t: Throwable =>
              killTaskManagerFatal(
                "Unable to start TaskManager components after registering at JobManager", t)
          }
        }

      case RefuseRegistration(reason) =>
        if (currentJobManager.isEmpty) {
          log.error(s"The registration at JobManager $jobManagerAkkaURL was refused, " +
            s"because: $reason. Retrying later...")

          if(jobManagerAkkaURL.isDefined) {
            // try the registration again after some time
            val delay: FiniteDuration = new FiniteDuration(
              config.getRefusedRegistrationPause().getSize(),
              config.getRefusedRegistrationPause().getUnit())
            val deadline: Option[Deadline] = Option(config.getMaxRegistrationDuration())
              .map {
                duration => new FiniteDuration(duration.getSize(), duration.getUnit()) +
                  delay fromNow
              }

            // start a new registration run
            currentRegistrationRun = UUID.randomUUID()

            scheduledTaskManagerRegistration.foreach(_.cancel())

            scheduledTaskManagerRegistration = Option(
              context.system.scheduler.scheduleOnce(delay) {
                self ! decorateMessage(
                  TriggerTaskManagerRegistration(
                    jobManagerAkkaURL.get,
                    new FiniteDuration(
                      config.getInitialRegistrationPause().getSize(),
                      config.getInitialRegistrationPause().getUnit()),
                    deadline,
                    1,
                    currentRegistrationRun)
                )
              }(context.dispatcher))
          }
        } else {
          // ignore RefuseRegistration messages which arrived after AcknowledgeRegistration
          if (sender() == currentJobManager.orNull) {
            log.warn(s"Received 'RefuseRegistration' from the JobManager (${sender().path})" +
              s" even though this TaskManager is already registered there.")
          } else {
            log.warn(s"Ignoring 'RefuseRegistration' from unrelated " +
              s"JobManager (${sender().path})")
          }
        }

      case _ => unhandled(message)
    }
    }



### task operation
    private def submitTask(tdd: TaskDeploymentDescriptor): Unit = {
    try {
      // grab some handles and sanity check on the fly
      val jobManagerActor = currentJobManager match {
        case Some(jm) => jm
        case None =>
          throw new IllegalStateException("TaskManager is not associated with a JobManager.")
      }
      val libCache = libraryCacheManager match {
        case Some(manager) => manager
        case None => throw new IllegalStateException("There is no valid library cache manager.")
      }

      val slot = tdd.getTargetSlotNumber
      if (slot < 0 || slot >= numberOfSlots) {
        throw new IllegalArgumentException(s"Target slot $slot does not exist on TaskManager.")
      }

      val (checkpointResponder,
        partitionStateChecker,
        resultPartitionConsumableNotifier,
        taskManagerConnection) = connectionUtils match {
        case Some(x) => x
        case None => throw new IllegalStateException("The connection utils have not been " +
                                                       "initialized.")
      }

      // create the task. this does not grab any TaskManager resources or download
      // and libraries - the operation does not block

      val jobManagerGateway = new AkkaActorGateway(jobManagerActor, leaderSessionID.orNull)

      val jobInformation = try {
        tdd.getSerializedJobInformation.deserializeValue(getClass.getClassLoader)
      } catch {
        case e @ (_: IOException | _: ClassNotFoundException) =>
          throw new IOException("Could not deserialize the job information.", e)
      }

      val taskInformation = try {
        tdd.getSerializedTaskInformation.deserializeValue(getClass.getClassLoader)
      } catch {
        case e@(_: IOException | _: ClassNotFoundException) =>
          throw new IOException("Could not deserialize the job vertex information.", e)
      }

      val taskMetricGroup = taskManagerMetricGroup.addTaskForJob(
        jobInformation.getJobId,
        jobInformation.getJobName,
        taskInformation.getJobVertexId,
        tdd.getExecutionAttemptId,
        taskInformation.getTaskName,
        tdd.getSubtaskIndex,
        tdd.getAttemptNumber)

      val inputSplitProvider = new TaskInputSplitProvider(
        jobManagerGateway,
        jobInformation.getJobId,
        taskInformation.getJobVertexId,
        tdd.getExecutionAttemptId,
        new FiniteDuration(
          config.getTimeout().getSize(),
          config.getTimeout().getUnit()))

      val task = new Task(
        jobInformation,
        taskInformation,
        tdd.getExecutionAttemptId,
        tdd.getAllocationId,
        tdd.getSubtaskIndex,
        tdd.getAttemptNumber,
        tdd.getProducedPartitions,
        tdd.getInputGates,
        tdd.getTargetSlotNumber,
        tdd.getTaskStateHandles,
        memoryManager,
        ioManager,
        network,
        bcVarManager,
        taskManagerConnection,
        inputSplitProvider,
        checkpointResponder,
        libCache,
        fileCache,
        config,
        taskMetricGroup,
        resultPartitionConsumableNotifier,
        partitionStateChecker,
        context.dispatcher)

      log.info(s"Received task ${task.getTaskInfo.getTaskNameWithSubtasks()}")

      val execId = tdd.getExecutionAttemptId
      // add the task to the map
      val prevTask = runningTasks.put(execId, task)
      if (prevTask != null) {
        // already have a task for that ID, put if back and report an error
        runningTasks.put(execId, prevTask)
        throw new IllegalStateException("TaskManager already contains a task for id " + execId)
      }
      
      // 启动task
      task.startTaskThread()

      sender ! decorateMessage(Acknowledge.get())
    }
    catch {
      case t: Throwable =>
        log.error("SubmitTask failed", t)
        sender ! decorateMessage(Status.Failure(t))
    }
    }

### Task 
- task执行的核心方法

		public void run() {

		// ----------------------------
		//  初始化状态Initial State transition
		// ----------------------------
		while (true) {
			ExecutionState current = this.executionState;
			if (current == ExecutionState.CREATED) {
				if (transitionState(ExecutionState.CREATED, ExecutionState.DEPLOYING)) {
					// success, we can start our work
					break;
				}
			}
			else if (current == ExecutionState.FAILED) {
				// we were immediately failed. tell the TaskManager that we reached our final state
				notifyFinalState();
				if (metrics != null) {
					metrics.close();
				}
				return;
			}
			else if (current == ExecutionState.CANCELING) {
				if (transitionState(ExecutionState.CANCELING, ExecutionState.CANCELED)) {
					// we were immediately canceled. tell the TaskManager that we reached our final state
					notifyFinalState();
					if (metrics != null) {
						metrics.close();
					}
					return;
				}
			}
			else {
				if (metrics != null) {
					metrics.close();
				}
				throw new IllegalStateException("Invalid state for beginning of operation of task " + this + '.');
			}
		}

		// 所有资源在最后都需要关闭
		Map<String, Future<Path>> distributedCacheEntries = new HashMap<>();
		AbstractInvokable invokable = null;

		try {
			// ----------------------------
			//  Task Bootstrap - We periodically
			//  check for canceling as a shortcut
			// ----------------------------

			// activate safety net for task thread
			LOG.info("Creating FileSystem stream leak safety net for task {}", this);
			FileSystemSafetyNet.initializeSafetyNetForThread();

			blobService.getPermanentBlobService().registerJob(jobId);

			// first of all, get a user-code classloader
			// this may involve downloading the job's JAR files and/or classes
			LOG.info("Loading JAR files for task {}.", this);

			userCodeClassLoader = createUserCodeClassloader();
			final ExecutionConfig executionConfig = serializedExecutionConfig.deserializeValue(userCodeClassLoader);

			if (executionConfig.getTaskCancellationInterval() >= 0) {
				// override task cancellation interval from Flink config if set in ExecutionConfig
				taskCancellationInterval = executionConfig.getTaskCancellationInterval();
			}

			if (executionConfig.getTaskCancellationTimeout() >= 0) {
				// override task cancellation timeout from Flink config if set in ExecutionConfig
				taskCancellationTimeout = executionConfig.getTaskCancellationTimeout();
			}

			if (isCanceledOrFailed()) {
				throw new CancelTaskException();
			}

			// ----------------------------------------------------------------
			// register the task with the network stack
			// this operation may fail if the system does not have enough
			// memory to run the necessary data exchanges
			// the registration must also strictly be undone
			// ----------------------------------------------------------------

			LOG.info("Registering task at network: {}.", this);

			network.registerTask(this);

			// add metrics for buffers
			this.metrics.getIOMetricGroup().initializeBufferMetrics(this);

			// register detailed network metrics, if configured
			if (taskManagerConfig.getConfiguration().getBoolean(TaskManagerOptions.NETWORK_DETAILED_METRICS)) {
				// similar to MetricUtils.instantiateNetworkMetrics() but inside this IOMetricGroup
				MetricGroup networkGroup = this.metrics.getIOMetricGroup().addGroup("Network");
				MetricGroup outputGroup = networkGroup.addGroup("Output");
				MetricGroup inputGroup = networkGroup.addGroup("Input");

				// output metrics
				for (int i = 0; i < producedPartitions.length; i++) {
					ResultPartitionMetrics.registerQueueLengthMetrics(
						outputGroup.addGroup(i), producedPartitions[i]);
				}

				for (int i = 0; i < inputGates.length; i++) {
					InputGateMetrics.registerQueueLengthMetrics(
						inputGroup.addGroup(i), inputGates[i]);
				}
			}

			// next, kick off the background copying of files for the distributed cache
			try {
				for (Map.Entry<String, DistributedCache.DistributedCacheEntry> entry :
						DistributedCache.readFileInfoFromConfig(jobConfiguration)) {
					LOG.info("Obtaining local cache file for '{}'.", entry.getKey());
					Future<Path> cp = fileCache.createTmpFile(entry.getKey(), entry.getValue(), jobId);
					distributedCacheEntries.put(entry.getKey(), cp);
				}
			}
			catch (Exception e) {
				throw new Exception(
					String.format("Exception while adding files to distributed cache of task %s (%s).", taskNameWithSubtask, executionId),
					e);
			}

			if (isCanceledOrFailed()) {
				throw new CancelTaskException();
			}

			// ----------------------------------------------------------------
			//  call the user code initialization methods
			// ----------------------------------------------------------------

			TaskKvStateRegistry kvStateRegistry = network.createKvStateTaskRegistry(jobId, getJobVertexId());

			Environment env = new RuntimeEnvironment(
				jobId,
				vertexId,
				executionId,
				executionConfig,
				taskInfo,
				jobConfiguration,
				taskConfiguration,
				userCodeClassLoader,
				memoryManager,
				ioManager,
				broadcastVariableManager,
				taskStateManager,
				accumulatorRegistry,
				kvStateRegistry,
				inputSplitProvider,
				distributedCacheEntries,
				producedPartitions,
				inputGates,
				network.getTaskEventDispatcher(),
				checkpointResponder,
				taskManagerConfig,
				metrics,
				this);

			// now load and instantiate the task's invokable code
			invokable = loadAndInstantiateInvokable(userCodeClassLoader, nameOfInvokableClass, env);

			// we must make strictly sure that the invokable is accessible to the cancel() call
			// by the time we switched to running.
			this.invokable = invokable;

			// switch to the RUNNING state, if that fails, we have been canceled/failed in the meantime
			if (!transitionState(ExecutionState.DEPLOYING, ExecutionState.RUNNING)) {
				throw new CancelTaskException();
			}

			// notify everyone that we switched to running
			notifyObservers(ExecutionState.RUNNING, null);
			taskManagerActions.updateTaskExecutionState(new TaskExecutionState(jobId, executionId, ExecutionState.RUNNING));

			// make sure the user code classloader is accessible thread-locally
			executingThread.setContextClassLoader(userCodeClassLoader);

			// 真正的task执行
			invokable.invoke();

			// make sure, we enter the catch block if the task leaves the invoke() method due
			// to the fact that it has been canceled
			if (isCanceledOrFailed()) {
				throw new CancelTaskException();
			}

			// ----------------------------------------------------------------
			//  finalization of a successful execution
			// ----------------------------------------------------------------

			// finish the produced partitions. if this fails, we consider the execution failed.
			for (ResultPartition partition : producedPartitions) {
				if (partition != null) {
					partition.finish();
				}
			}

			// try to mark the task as finished
			// if that fails, the task was canceled/failed in the meantime
			if (transitionState(ExecutionState.RUNNING, ExecutionState.FINISHED)) {
				notifyObservers(ExecutionState.FINISHED, null);
			}
			else {
				throw new CancelTaskException();
			}
		}
		catch (Throwable t) {
			。。。
		}
		finally {
			。。。
		}
	}

### StreamTask 
	
		public final void invoke() throws Exception {

		boolean disposed = false;
		try {
			// -------- 初始化 ---------
			LOG.debug("Initializing {}.", getName());

			asyncOperationsThreadPool = Executors.newCachedThreadPool();

			CheckpointExceptionHandlerFactory cpExceptionHandlerFactory = createCheckpointExceptionHandlerFactory();

			synchronousCheckpointExceptionHandler = cpExceptionHandlerFactory.createCheckpointExceptionHandler(
				getExecutionConfig().isFailTaskOnCheckpointError(),
				getEnvironment());

			asynchronousCheckpointExceptionHandler = new AsyncCheckpointExceptionHandler(this);

			stateBackend = createStateBackend();
			checkpointStorage = stateBackend.createCheckpointStorage(getEnvironment().getJobID());

			accumulatorMap = getEnvironment().getAccumulatorRegistry().getUserMap();

			// if the clock is not already set, then assign a default TimeServiceProvider
			if (timerService == null) {
				ThreadFactory timerThreadFactory =
					new DispatcherThreadFactory(TRIGGER_THREAD_GROUP, "Time Trigger for " + getName());

				timerService = new SystemProcessingTimeService(this, getCheckpointLock(), timerThreadFactory);
			}

			operatorChain = new OperatorChain<>(this, streamRecordWriters);
			headOperator = operatorChain.getHeadOperator();

			// task specific initialization
			init();

			// save the work of reloading state, etc, if the task is already canceled
			if (canceled) {
				throw new CancelTaskException();
			}

			// -------- Invoke --------
			LOG.debug("Invoking {}", getName());

			// 打开所有operator
			synchronized (lock) {

				// both the following operations are protected by the lock
				// so that we avoid race conditions in the case that initializeState()
				// registers a timer, that fires before the open() is called.

				initializeState();
				openAllOperators();
			}

			// final check to exit early before starting to run
			if (canceled) {
				throw new CancelTaskException();
			}

			// let the task do its work
			isRunning = true;
            //task真正执行逻辑
			run();

			if (canceled) {
				throw new CancelTaskException();
			}

			LOG.debug("Finished task {}", getName());
			//关闭task所有资源
			synchronized (lock) {
				closeAllOperators();
				timerService.quiesce();
				isRunning = false;
			}
			
			timerService.awaitPendingAfterQuiesce();
			LOG.debug("Closed operators for task {}", getName());
			operatorChain.flushOutputs();
			tryDisposeAllOperators();
			disposed = true;
		}
		finally {
			。。。
		}
	}


### operator的产生，存储在operatorChain中，由StreamTask产生

		public OperatorChain(
			StreamTask<OUT, OP> containingTask,
			List<StreamRecordWriter<SerializationDelegate<StreamRecord<OUT>>>> streamRecordWriters) {

		final ClassLoader userCodeClassloader = containingTask.getUserCodeClassLoader();
		final StreamConfig configuration = containingTask.getConfiguration();

		headOperator = configuration.getStreamOperator(userCodeClassloader);

		// we read the chained configs, and the order of record writer registrations by output name
		Map<Integer, StreamConfig> chainedConfigs = configuration.getTransitiveChainedTaskConfigsWithSelf(userCodeClassloader);

		// create the final output stream writers
		// we iterate through all the out edges from this job vertex and create a stream output
		List<StreamEdge> outEdgesInOrder = configuration.getOutEdgesInOrder(userCodeClassloader);
		Map<StreamEdge, RecordWriterOutput<?>> streamOutputMap = new HashMap<>(outEdgesInOrder.size());
		this.streamOutputs = new RecordWriterOutput<?>[outEdgesInOrder.size()];

		// from here on, we need to make sure that the output writers are shut down again on failure
		boolean success = false;
		try { 
            //每个edge对应一个output
			for (int i = 0; i < outEdgesInOrder.size(); i++) {
				StreamEdge outEdge = outEdgesInOrder.get(i);

				RecordWriterOutput<?> streamOutput = createStreamOutput(
					streamRecordWriters.get(i),
					outEdge,
					chainedConfigs.get(outEdge.getSourceId()),
					containingTask.getEnvironment());

				this.streamOutputs[i] = streamOutput;
				streamOutputMap.put(outEdge, streamOutput);
			}

			// we create the chain of operators and grab the collector that leads into the chain
			List<StreamOperator<?>> allOps = new ArrayList<>(chainedConfigs.size());
			this.chainEntryPoint = createOutputCollector(
				containingTask,
				configuration,
				chainedConfigs,
				userCodeClassloader,
				streamOutputMap,
				allOps);

			if (headOperator != null) {
				WatermarkGaugeExposingOutput<StreamRecord<OUT>> output = getChainEntryPoint();
				headOperator.setup(containingTask, configuration, output);

				headOperator.getMetricGroup().gauge(MetricNames.IO_CURRENT_OUTPUT_WATERMARK, output.getWatermarkGauge());
			}

			// 把headOperator加在list最后，因为createOutputCollector方法递归执行，所以operator在list中的顺序和逻辑顺序相反，之后执行的时候从list尾部开始执行
			allOps.add(headOperator);

			this.allOperators = allOps.toArray(new StreamOperator<?>[allOps.size()]);

			success = true;
		}
		finally {
			。。。
		}

	}


### createOutputCollector 看下如何递归调用的

		private <T> WatermarkGaugeExposingOutput<StreamRecord<T>> createOutputCollector(
			StreamTask<?, ?> containingTask,
			StreamConfig operatorConfig,
			Map<Integer, StreamConfig> chainedConfigs,
			ClassLoader userCodeClassloader,
			Map<StreamEdge, RecordWriterOutput<?>> streamOutputs,
			List<StreamOperator<?>> allOperators) {
		List<Tuple2<WatermarkGaugeExposingOutput<StreamRecord<T>>, StreamEdge>> allOutputs = new ArrayList<>(4);

		// create collectors for the network outputs
		for (StreamEdge outputEdge : operatorConfig.getNonChainedOutputs(userCodeClassloader)) {
			@SuppressWarnings("unchecked")
			RecordWriterOutput<T> output = (RecordWriterOutput<T>) streamOutputs.get(outputEdge);

			allOutputs.add(new Tuple2<>(output, outputEdge));
		}

		// 递归截止条件是:最后一个operator的operatorConfig.getChainedOutputs为空
		for (StreamEdge outputEdge : operatorConfig.getChainedOutputs(userCodeClassloader)) {
            // 获得下游operator id
			int outputId = outputEdge.getTargetId();
            // 获得下游operator的streamConfig
			StreamConfig chainedOpConfig = chainedConfigs.get(outputId);
            // 根据下游operator的config创建ChainedOperator，返回chain的output
			WatermarkGaugeExposingOutput<StreamRecord<T>> output = createChainedOperator(
				containingTask,
				chainedOpConfig,
				chainedConfigs,
				userCodeClassloader,
				streamOutputs,
				allOperators,
				outputEdge.getOutputTag());
			allOutputs.add(new Tuple2<>(output, outputEdge));
		}
	}

### createChainedOperator
	private <IN, OUT> WatermarkGaugeExposingOutput<StreamRecord<IN>> createChainedOperator(
			StreamTask<?, ?> containingTask,
			StreamConfig operatorConfig,
			Map<Integer, StreamConfig> chainedConfigs,
			ClassLoader userCodeClassloader,
			Map<StreamEdge, RecordWriterOutput<?>> streamOutputs,
			List<StreamOperator<?>> allOperators,
			OutputTag<IN> outputTag) {
		// create the output that the operator writes to first. this may recursively create more operators
		WatermarkGaugeExposingOutput<StreamRecord<OUT>> chainedOperatorOutput = createOutputCollector(
			containingTask,
			operatorConfig,
			chainedConfigs,
			userCodeClassloader,
			streamOutputs,
			allOperators);

		// now create the operator and give it the output collector to write its output to
		OneInputStreamOperator<IN, OUT> chainedOperator = operatorConfig.getStreamOperator(userCodeClassloader);

		chainedOperator.setup(containingTask, operatorConfig, chainedOperatorOutput);

		allOperators.add(chainedOperator);

		WatermarkGaugeExposingOutput<StreamRecord<IN>> currentOperatorOutput;
		if (containingTask.getExecutionConfig().isObjectReuseEnabled()) {
			currentOperatorOutput = new ChainingOutput<>(chainedOperator, this, outputTag);
		}
		else {
			TypeSerializer<IN> inSerializer = operatorConfig.getTypeSerializerIn1(userCodeClassloader);
			currentOperatorOutput = new CopyingChainingOutput<>(chainedOperator, inSerializer, outputTag, this);
		}

		chainedOperator.getMetricGroup().gauge(MetricNames.IO_CURRENT_INPUT_WATERMARK, currentOperatorOutput.getWatermarkGauge());
		chainedOperator.getMetricGroup().gauge(MetricNames.IO_CURRENT_OUTPUT_WATERMARK, chainedOperatorOutput.getWatermarkGauge());

		return currentOperatorOutput;
	}

