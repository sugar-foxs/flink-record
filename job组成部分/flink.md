https://flink.apache.org/usecases.html

## EventTime
### Processing time
- Processing time指的是执行相应操作的机器的系统时间。
### Event time
- Event time是每个单独事件发生在其产生设备上的时间。
- 时间水印
- 在一个完美的世界，事件时间处理会产生完全一致和确定的结果，不管事件是什么时候到达的，或它们的顺序。但是，除非事件已知按顺序到达（按时间戳），否则事件时间处理会在等待无序事件时产生一些延迟。 由于只能等待一段有限的时间，因此限制了确定性事件时间应用程序的可能性。
- 假设所有数据都已到达，事件时间操作将按预期运行，即使在处理无序或延迟事件或重新处理历史数据时也会产生正确且一致的结果。
### Ingestion time
- Ingestion time是事件进入flink的时间


ClusterClient:MiniClusterClient,RestClustrClient,StandaloneClusterClient

本地模式：环境LocalStreamEnvironment execute()->MiniCluster.executeJobBlocking(jobGraph)
集群模式：StreamContextEnvironment execute()->client submitJob()