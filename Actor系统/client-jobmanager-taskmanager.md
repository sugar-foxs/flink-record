无论程序是用DataStream还是DataSet编写的，Client 都会转成JobGraph,发送至jobManagaer,jobManager将JobGraph转换成ExecutionGraph，


jobManager和taskManager用actor模式通信

### jobManager接收到JobGraph的处理方法：

    private def submitJob(jobGraph: JobGraph, jobInfo: JobInfo, isRecovery: Boolean = false): Unit = {
    if (jobGraph == null) {
      jobInfo.notifyClients(
        decorateMessage(JobResultFailure(
          new SerializedThrowable(
            new JobSubmissionException(null, "JobGraph must not be null.")))))
    }
    else {
      val jobId = jobGraph.getJobID
      val jobName = jobGraph.getName
      var executionGraph: ExecutionGraph = null
    
      log.info(s"Submitting job $jobId ($jobName)" + (if (isRecovery) " (Recovery)" else "") + ".")
    
      try {
        // 先设置用户代码库，为了在不成功的情况想把已经上传到jars删除
        try {
          libraryCacheManager.registerJob(
            jobGraph.getJobID, jobGraph.getUserJarBlobKeys, jobGraph.getClasspaths)
        }
        catch {
          case t: Throwable =>
            throw new JobSubmissionException(jobId,
              "Cannot set up the user code libraries: " + t.getMessage, t)
        }
    
        val userCodeLoader = libraryCacheManager.getClassLoader(jobGraph.getJobID)
        if (userCodeLoader == null) {
          throw new JobSubmissionException(jobId,
            "The user code class loader could not be initialized.")
        }
    
        //jobGraph中task数量为0
        if (jobGraph.getNumberOfVertices == 0) {
          throw new JobSubmissionException(jobId, "The given job is empty")
        }
    
        //重启策略
        val restartStrategy =
          Option(jobGraph.getSerializedExecutionConfig()
            .deserializeValue(userCodeLoader)
            .getRestartStrategy())
            .map(RestartStrategyFactory.createRestartStrategy)
            .filter(p => p != null) match {
            case Some(strategy) => strategy
            case None => restartStrategyFactory.createRestartStrategy()
          }
    
        log.info(s"Using restart strategy $restartStrategy for $jobId.")
    
        //加入group中
        val jobMetrics = jobManagerMetricGroup.addJob(jobGraph)
    
        val numSlots = scheduler.getTotalNumberOfSlots()
    
        //是否需要注册新graph
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
    
        //创建executionGraph
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
          Time.milliseconds(allocationTimeout),
          log.logger)
        
        if (registerNewGraph) {
          currentJobs.put(jobGraph.getJobID, (executionGraph, jobInfo))
        }
    
        // 注册job状态监听器
        executionGraph.registerJobStatusListener(
          new StatusListenerMessenger(self, leaderSessionID.orNull))
    
        jobInfo.clients foreach {
          // the sender wants to be notified about state changes
          case (client, ListeningBehaviour.EXECUTION_RESULT_AND_STATE_CHANGES) =>
            val listener  = new StatusListenerMessenger(client, leaderSessionID.orNull)
            executionGraph.registerExecutionListener(listener)
            executionGraph.registerJobStatusListener(listener)
          case _ => // do nothing
        }
    
      } catch {
        case t: Throwable =>
          log.error(s"Failed to submit job $jobId ($jobName)", t)
    
          libraryCacheManager.unregisterJob(jobId)
          blobServer.cleanupJob(jobId, true)
          currentJobs.remove(jobId)
    
          if (executionGraph != null) {
            executionGraph.failGlobal(t)
          }
    
          val rt: Throwable = if (t.isInstanceOf[JobExecutionException]) {
            t
          } else {
            new JobExecutionException(jobId, s"Failed to submit job $jobId ($jobName)", t)
          }
    
          jobInfo.notifyClients(
            decorateMessage(JobResultFailure(new SerializedThrowable(rt))))
          return
      }
    
      // 恢复措施，将jobGraph写入SubmittedJobGraphStore（异步）
      future {
        try {
          if (isRecovery) {
            //恢复至最新的checkpoint
            executionGraph.restoreLatestCheckpointedState(false, false)
          }
          else {
            // 载入保存点
            val savepointSettings = jobGraph.getSavepointRestoreSettings
            if (savepointSettings.restoreSavepoint()) {
              try {
                val savepointPath = savepointSettings.getRestorePath()
                val allowNonRestored = savepointSettings.allowNonRestoredState()
    
                executionGraph.getCheckpointCoordinator.restoreSavepoint(
                  savepointPath, 
                  allowNonRestored,
                  executionGraph.getAllVertices,
                  executionGraph.getUserClassLoader
                )
              } catch {
                case e: Exception =>
                  jobInfo.notifyClients(
                    decorateMessage(JobResultFailure(new SerializedThrowable(e))))
                  throw new SuppressRestartsException(e)
              }
            }
    
            try {
              submittedJobGraphs.putJobGraph(new SubmittedJobGraph(jobGraph, jobInfo))
            } catch {
              case t: Throwable =>
                // Don't restart the execution if this fails. Otherwise, the
                // job graph will skip ZooKeeper in case of HA.
                jobInfo.notifyClients(
                  decorateMessage(JobResultFailure(new SerializedThrowable(t))))
                throw new SuppressRestartsException(t)
            }
          }
          //通知clients任务提交成功
          jobInfo.notifyClients(
            decorateMessage(JobSubmitSuccess(jobGraph.getJobID)))
    
          if (leaderElectionService.hasLeadership) {
            // There is a small chance that multiple job managers schedule the same job after if
            // they try to recover at the same time. This will eventually be noticed, but can not be
            // ruled out from the beginning.
    
            // NOTE: Scheduling the job for execution is a separate action from the job submission.
            // The success of submitting the job must be independent from the success of scheduling
            // the job.
            log.info(s"Scheduling job $jobId ($jobName).")
    
            executionGraph.scheduleForExecution()
          } else {
            // Remove the job graph. Otherwise it will be lingering around and possibly removed from
            // ZooKeeper by this JM.
            self ! decorateMessage(RemoveJob(jobId, removeJobFromStateBackend = false))
    
            log.warn(s"Submitted job $jobId, but not leader. The other leader needs to recover " +
              "this. I am not scheduling the job for execution.")
          }
        } catch {
          case t: Throwable => try {
            executionGraph.failGlobal(t)
          } catch {
            case tt: Throwable =>
              log.error("Error while marking ExecutionGraph as failed.", tt)
          }
        }
      }(context.dispatcher)
    }
  }