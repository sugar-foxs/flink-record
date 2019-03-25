背压监控
通过定时收集堆栈信息，
通过栈中调用org.apache.flink.runtime.io.network.buffer.LocalBufferPool.requestBufferBuilderBlocking
的比例来表示背压级别，<0.1 OK,<0.5 LOW, 其他High

通过调用Thread.getStackTrace获取栈信息，得到StackTraceElement[]数组，里面包含了类名，方法名，字段名，行数。默认最多取3层。

ratio = 类名+方法名是requestBufferBuilderBlocking的StackTraceElement / StackTraceElement[]的size