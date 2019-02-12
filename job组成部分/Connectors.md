# Connectors

- flink内置的source，sink。从文件、网络、collections、iterators获取，往文件、标准输出、网络写数据。
- flink支持连接的第三方系统有:kafka（source/sink）、Cassandra(sink)、Amazon Kinesis Streams(source/sink)、Elasticsearch(sink)、hadoop(sink)、RabbitMQ(source/sink)、NiFi(source/sink)、Twitter Streaming API(source)
- Apache Bahir进行的扩展有：ActiveMQ（source/sink）、Flume(sink)、Redis(sink)、Akka(sink)、Netty(source)
- 其他连接flink的方法：异步IO、Queryable state

## 1 kafka

- Flink1.7之后，有一个kafka connector不是与特定的kafka版本连接，而是在以后的版本中保持与kafka最新版本连接。
- 可以设置消费的起始offset，也可以为每个分区制定特定的offset。但是这个offsset对从失败状态恢复的任务不起作用，因为使用的是savepoint或者checkpoint中的offset。
- 失败任务从最新成功的Checkpoint恢复，checkpoint的间隔决定了需要回退到多久之前。如果checkpoint没有设置，kafka consumer会将offset提交到zk。
- 支持发现动态创建的kafka分区，并使用一次性保证来消费它们。 在初始检索分区元数据之后（即，当作业开始运行时）发现的所有分区将从最早可能的偏移量中消费。通过配置flink.partition-discovery.interval-millis启用。
- 高版本flink也支持了topic发现，在任务运行之后，发现符合正则表达式的topic名字。通过配置flink.partition-discovery.interval-millis启用。
- 允许配置如何将偏移提交回Kafka brokers（或0.8中的Zookeeper）的行为,这个offset对容错没有用，只是用来监控消费者进程。checkpoint是否启用，配置提交offset的行为的方法是不同的。
  - checkpoint没启用的情况下，设置enable.auto.commit来启用/停用offset提交。
  - checkpoint启用的情况下，Flink Kafka Consumer将在检查点完成时提交存储在检查点状态中的偏移量。 这可确保Kafka brokers中的偏移量与检查点状态中的偏移量一致。setCommitOffsetsOnCheckpoints（true/false）来启用/停用offset提交，默认是true。
- 在许多情况下，记录的时间戳（显式或隐式）嵌入记录本身。 另外，用户可能想要周期性地或以不规则的方式发出水印，例如包含当前事件时间水印的特殊记录Kafka流。 对于这些情况，Flink Kafka Consumer允许指定AssignerWithPeriodicWatermarks或AssignerWithPunctuatedWatermarks。在内部，每个Kafka分区执行一个分配器实例。 当指定这样的分配器时，对于从Kafka读取的每个记录，调用extractTimestamp来为记录分配时间戳，并调用getCurrentWatermark()(for periodic)或checkAndGetNextWatermark(for punctuated)方法以确定是否应该发出新的水印以及使用哪个时间戳。
