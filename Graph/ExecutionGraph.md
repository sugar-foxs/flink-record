# ExecutionGraph

jobManager在接收到submitJob消息之后，会先根据jobGraph生成ExecutionGraph。

1，使用ExecutionGraphBuilder的buildGraph方法生成ExecutionGraph；

```
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
```

2，深入到buildGraph方法，如果Execution不存在，new一个新的Execution;

```
executionGraph = (prior != null) ? prior :
				new ExecutionGraph(
					jobInformation,
					futureExecutor,
					ioExecutor,
					rpcTimeout,
					restartStrategy,
					failoverStrategy,
					slotProvider,
					classLoader,
					blobWriter,
					resultPartitionLocationTrackerProxy,
					allocationTimeout,
					metrics,
					jobManagerConfig);
```

3，设置一些基础属性:allowQueuedScheduling,jsonPlan;

```
executionGraph.setQueuedSchedulingAllowed(jobGraph.getAllowQueuedScheduling());
executionGraph.setJsonPlan(JsonPlanGenerator.generatePlan(jobGraph));
```

4，attachJobGraph，生成Graph的节点和边

```
// topologically sort the job vertices and attach the graph to the existing one
		List<JobVertex> sortedTopology = jobGraph.getVerticesSortedTopologicallyFromSources();
		if (log.isDebugEnabled()) {
			log.debug("Adding {} vertices from job graph {} ({}).", sortedTopology.size(), jobName, jobId);
		}
		executionGraph.attachJobGraph(sortedTopology);
```

深入到attachJobGraph方法，将JobVertex封装成ExecutionJobVertex,然后再用edge将节点连起来；

```
public void attachJobGraph(List<JobVertex> topologicallySorted) throws JobException {

		final ArrayList<ExecutionJobVertex> newExecJobVertices = new ArrayList<>(topologicallySorted.size());
		inal long createTimestamp = System.currentTimeMillis();

		for (JobVertex jobVertex : topologicallySorted) {

			if (jobVertex.isInputVertex() && !jobVertex.isStoppable()) {
				this.isStoppable = false;
			}

			// 在新建ExecutionJobVertex时已经把中间结果集都初始化好。
			ExecutionJobVertex ejv = new ExecutionJobVertex(
				this,
				jobVertex,
				1,
				rpcTimeout,
				globalModVersion,
				createTimestamp);
            //把节点用edge相连
			ejv.connectToPredecessors(this.intermediateResults);
			
			//ExecutionJobVertex建好并与input建立好edge,存入map中
			ExecutionJobVertex previousTask = this.tasks.putIfAbsent(jobVertex.getID(), ejv);
			for (IntermediateResult res : ejv.getProducedDataSets()) {
				IntermediateResult previousDataSet = 						this.intermediateResults.putIfAbsent(res.getId(), res);
			}
			this.verticesInCreationOrder.add(ejv);
			this.numVerticesTotal += ejv.getParallelism();
			newExecJobVertices.add(ejv);
		}

		terminationFuture = new CompletableFuture<>();
		failoverStrategy.notifyNewVertices(newExecJobVertices);
	}
```

IntermediateResult 对应一个 Job Edge 的输出下游结果集，一个 IntermediateResult 包含多个 IntermediateResultPartition，一个 IntermediateResultPartition 对应一个并行任务 ExecutionVertex 的输出结果。

深入到connectToPredecessors，看看是如何连接ExecutionJobVertex的；

```
public void connectToPredecessors(Map<IntermediateDataSetID, IntermediateResult> intermediateDataSets) throws JobException {
		//获取节点所有输入边
		List<JobEdge> inputs = jobVertex.getInputs();
		for (int num = 0; num < inputs.size(); num++) {
			JobEdge edge = inputs.get(num);
			//取出JobEdge的source IntermediateResult
			IntermediateResult ires = intermediateDataSets.get(edge.getSourceId());

			this.inputs.add(ires);
			//将当前vertex作为consumer注册到IntermediateResult的每个IntermediateResultPartition
			int consumerIndex = ires.registerConsumer();

			for (int i = 0; i < parallelism; i++) {
				ExecutionVertex ev = taskVertices[i];
				//为每个ExecutionVertex建立到具体IntermediateResultPartition的ExecutionEdge
				ev.connectSource(num, ires, edge, consumerIndex);
			}
		}
	}
```

深入到connectSource方法，

```
public void connectSource(int inputNumber, IntermediateResult source, JobEdge edge, int consumerNumber) {

		final DistributionPattern pattern = edge.getDistributionPattern();
		final IntermediateResultPartition[] sourcePartitions = source.getPartitions();

		ExecutionEdge[] edges;

		switch (pattern) {
			case POINTWISE:
				edges = connectPointwise(sourcePartitions, inputNumber);
				break;

			case ALL_TO_ALL:
				edges = connectAllToAll(sourcePartitions, inputNumber);
				break;

			default:
				throw new RuntimeException("Unrecognized distribution pattern.");

		}

		this.inputEdges[inputNumber] = edges;

		// add the consumers to the source
		// for now (until the receiver initiated handshake is in place), we need to register the
		// edges as the execution graph
		for (ExecutionEdge ee : edges) {
			ee.getSource().addConsumer(ee, consumerNumber);
		}
	}
```

如果pattern是POINTWISE的话，看下connectPointwise方法：

```
private ExecutionEdge[] connectPointwise(IntermediateResultPartition[] sourcePartitions, int inputNumber) {
		//source节点subtask个数
		final int numSources = sourcePartitions.length;
		//subtasks并发度
		final int parallelism = getTotalNumberOfParallelSubtasks();

		// source节点subtask个数等于当前节点并发度的话，取sourcePartitions中subTaskIndex对应的partition，一一对应
		if (numSources == parallelism) {
			return new ExecutionEdge[] { new ExecutionEdge(sourcePartitions[subTaskIndex], this, inputNumber) };
		}
		else if (numSources < parallelism) { //如果并发度比partition个数多，那一个source会对应于多个task
			int sourcePartition;
			//当前节点并发度整除source节点subtask个数的情况下，每个source对应相同数目的task,假设有2个source，6个task，第3,4,5个task对应第1个source
			if (parallelism % numSources == 0) {
				// same number of targets per source
				int factor = parallelism / numSources;
				sourcePartition = subTaskIndex / factor;
			}
			else {
				// 比如有2个source,7个task,第0个source对应第0，1，2个task,第1个source对应第3，4，5，6个task
				float factor = ((float) parallelism) / numSources;
				sourcePartition = (int) (subTaskIndex / factor);
			}

			return new ExecutionEdge[] { new ExecutionEdge(sourcePartitions[sourcePartition], this, inputNumber) };
		}
		else {//多个source对应一个task
			//source节点subtask个数整除当前节点并发度的情况下，每个task有numSources / parallelism个source,假设有6个source，2个task,则第3，4，5个source对应第1个task
			if (numSources % parallelism == 0) {
				int factor = numSources / parallelism;
				int startIndex = subTaskIndex * factor;

				ExecutionEdge[] edges = new ExecutionEdge[factor];
				for (int i = 0; i < factor; i++) {
					edges[i] = new ExecutionEdge(sourcePartitions[startIndex + i], this, inputNumber);
				}
				return edges;
			}
			else {
				//比如有7个source, 2个task，第0，1，2个source对应第0个task，第3、4、5、6个source对应第1个task
				float factor = ((float) numSources) / parallelism;
				int start = (int) (subTaskIndex * factor);
				int end = (subTaskIndex == getTotalNumberOfParallelSubtasks() - 1) ?
						sourcePartitions.length :
						(int) ((subTaskIndex + 1) * factor);

				ExecutionEdge[] edges = new ExecutionEdge[end - start];
				for (int i = 0; i < edges.length; i++) {
					edges[i] = new ExecutionEdge(sourcePartitions[start + i], this, inputNumber);
				}

				return edges;
			}
		}
	}
```

如果pattern是ALL_TO_ALL的话，看下connectAllToAll方法：

```
private ExecutionEdge[] connectAllToAll(IntermediateResultPartition[] sourcePartitions, int inputNumber) {
		ExecutionEdge[] edges = new ExecutionEdge[sourcePartitions.length];
		for (int i = 0; i < sourcePartitions.length; i++) {
			IntermediateResultPartition irp = sourcePartitions[i];
			edges[i] = new ExecutionEdge(irp, this, inputNumber);
		}

		return edges;
	}
```

把所有的source都指向每个task。

5,配置checkpoint

```
JobCheckpointingSettings snapshotSettings = jobGraph.getCheckpointingSettings();
		if (snapshotSettings != null) {
			...
			executionGraph.enableCheckpointing(
				chkConfig.getCheckpointInterval(),
				chkConfig.getCheckpointTimeout(),
				chkConfig.getMinPauseBetweenCheckpoints(),
				chkConfig.getMaxConcurrentCheckpoints(),
				chkConfig.getCheckpointRetentionPolicy(),
				triggerVertices,
				ackVertices,
				confirmVertices,
				hooks,
				checkpointIdCounter,
				completedCheckpoints,
				rootBackend,
				checkpointStatsTracker);
		}
```

checkpoint之后专门讨论

6,设置metrics

```
metrics.gauge(RestartTimeGauge.METRIC_NAME, new RestartTimeGauge(executionGraph));
		metrics.gauge(DownTimeGauge.METRIC_NAME, new DownTimeGauge(executionGraph));
		metrics.gauge(UpTimeGauge.METRIC_NAME, new UpTimeGauge(executionGraph));
		metrics.gauge(NumberOfFullRestartsGauge.METRIC_NAME, new NumberOfFullRestartsGauge(executionGraph));

		executionGraph.getFailoverStrategy().registerMetrics(metrics);
```

