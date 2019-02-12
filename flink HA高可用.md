# Flink JobManager HA 高可用如何实现？

## 1，什么叫高可用？

- 单点故障是系统高可用的大敌，应该尽量在系统设计的过程中避免单点。方法论上，高可用保证的原则是“集群化”，或者叫“冗余”：只有一个单点，挂了服务会受影响；如果有冗余备份，挂了还有其他备份能够顶上。

- 有了冗余之后，还不够，每次出现故障需要人工介入恢复势必会增加系统的不可服务实践。所以，又往往是通过“自动故障转移”来实现系统的高可用。

- 所以，高可用是系统失败之后能够自动从故障中恢复，并继续为用户提供服务。

## 2，flink JobManager如何实现高可用？

- 1, 在Standalone集群模式下
  - JobManager高可用的方案是：Flink集群的任一时刻只有一个leading JobManager，并且有多个standby JobManager。当leader失败后，standby通过选举出一个JobManager作为新的leader。这个方案可以保证没有单点故障的问题。
  - Flink利用ZooKeeper在所有正在运行的JobManager实例之间进行分布式协调。 ZooKeeper是独立于Flink的服务，通过领导者选举和轻量级一致状态存储提供高度可靠的分布式协调。

- 2, 在yarn集群模式下
  - 我们不需要运行多个JobManager（ApplicationMaster）实例，只需要运行一个实例，如果失败了通过YARN来进行重启。









