# 重启策略

## 1，Fixed Delay Restart Strategy

Flink-conf.yaml中的配置：

restart-strategy.fixed-delay.attempts： 重启尝试次数

restart-strategy.fixed-delay.delay：失败之后到重启之间的时间，默认：akka.ask.timeout, 如果启用checkpoint，默认10s

# 2，Failure Rate Restart Strategy

restart-strategy.failure-rate.max-failures-per-interval：在internal内失败断的最大次数。

restart-strategy.failure-rate.failure-rate-interval：衡量失败率的单位时间。

restart-strategy.failure-rate.delay：两个连续重启操作之间的间隔时间。

# 3，No Restart Strategy

- 不重启

# 4， Fallback Restart Strategy

使用集群定义的重新启动策略。 这对于启用检查点的流式程序很有帮助。 默认情况下，如果没有定义其他重启策略，则选择固定延迟重启策略。

