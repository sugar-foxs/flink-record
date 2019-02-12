# Client

client提交job到JobManager的过程：

- ClusterClient的run方法，传入了JobGraph

```
public JobExecutionResult run(JobGraph jobGraph, ClassLoader classLoader) throws ProgramInvocationException {
		...
		try {
			this.lastJobExecutionResult = JobClient.submitJobAndWait(
				actorSystem,
				flinkConfig,
				highAvailabilityServices,
				jobGraph,
				timeout,
				printStatusDuringExecution,
				classLoader);

			return lastJobExecutionResult;
		} catch (JobExecutionException e) {
		...
		}
	}
```

- 追溯到JobClient的submitJobAndWait方法，调用了submitJob方法：

```
public static JobListeningContext submitJob(
			ActorSystem actorSystem,
			Configuration config,
			HighAvailabilityServices highAvailabilityServices,
			JobGraph jobGraph,
			FiniteDuration timeout,
			boolean sysoutLogUpdates,
			ClassLoader classLoader) {

		// 创建JobSubmissionClientActor，用于和JobManager交流，提交job等等
		Props jobClientActorProps = JobSubmissionClientActor.createActorProps(
	highAvailabilityServices.getJobManagerLeaderRetriever(HighAvailabilityServices.DEFAULT_JOB_ID),
			timeout,
			sysoutLogUpdates,
			config);

		ActorRef jobClientActor = actorSystem.actorOf(jobClientActorProps);
		//发送了SubmitJobAndWait消息
		Future<Object> submissionFuture = Patterns.ask(
				jobClientActor,
				new JobClientMessages.SubmitJobAndWait(jobGraph),
				new Timeout(AkkaUtils.INF_TIMEOUT()));

		return new JobListeningContext(
			jobGraph.getJobID(),
			submissionFuture,
			jobClientActor,
			timeout,
			classLoader,
			highAvailabilityServices);
	}
```

- 下面看下如何处理SubmitJobAndWait消息的。

```
public void handleCustomMessage(Object message) {
		// submit a job to the JobManager
		if (message instanceof SubmitJobAndWait) {
			if (this.client == null) {
				jobGraph = ((SubmitJobAndWait) message).jobGraph();
				if (jobGraph == null) {
					sender().tell(
						decorateMessage(new Status.Failure(new Exception("JobGraph is null"))),
						getSelf());
				} else {
					this.client = getSender();
					if (jobManager != null) {
					    //提交作业到JobManager
						tryToSubmitJob();
					}
				}
			} else {
				// 重复提交了
				String msg = "Received repeated 'SubmitJobAndWait'";
				LOG.error(msg);
				getSender().tell(
					decorateMessage(new Status.Failure(new Exception(msg))), ActorRef.noSender());

				terminate();
			}
		} else if ...
	}
```

- 深入到创建JobSubmissionClientActor的tryToSubmitJob方法中，

```
private void tryToSubmitJob() {
		final ActorGateway jobManagerGateway = new AkkaActorGateway(jobManager, leaderSessionID);
		final AkkaJobManagerGateway akkaJobManagerGateway = new AkkaJobManagerGateway(jobManagerGateway);
		final CompletableFuture<InetSocketAddress> blobServerAddressFuture = JobClient.retrieveBlobServerAddress(
			akkaJobManagerGateway,
			Time.milliseconds(timeout.toMillis()));
	    //上传jar包至jobManager
		final CompletableFuture<Void> jarUploadFuture = blobServerAddressFuture.thenAcceptAsync(
			(InetSocketAddress blobServerAddress) -> {
				try {
					ClientUtils.extractAndUploadJobGraphFiles(jobGraph, () -> new BlobClient(blobServerAddress, clientConfig));
				} catch (FlinkException e) {
					throw new CompletionException(e);
				}
			},
			getContext().dispatcher());
			
		jarUploadFuture
			.thenAccept(
				(Void ignored) -> {
				    //发送SubmitJob消息到JobManager
					jobManager.tell(
						decorateMessage(
							new JobManagerMessages.SubmitJob(
								jobGraph,
								ListeningBehaviour.EXECUTION_RESULT_AND_STATE_CHANGES)),
						getSelf());
					//提交超时
					getContext().system().scheduler().scheduleOnce(
						timeout,
						getSelf(),
						decorateMessage(JobClientMessages.getSubmissionTimeout()),
						getContext().dispatcher(),
						ActorRef.noSender());
				})
			.whenComplete(
				(Void ignored, Throwable throwable) -> {
					if (throwable != null) {
					    //提交失败
						getSelf().tell( 
							decorateMessage(new JobManagerMessages.JobResultFailure(
								new SerializedThrowable(ExceptionUtils.stripCompletionException(throwable)))),
							ActorRef.noSender());
					}
				});
	}
```

