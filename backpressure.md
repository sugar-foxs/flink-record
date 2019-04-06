背压监控
通过定时收集堆栈信息，调用Thread.getStackTrace获取栈信息，得到StackTraceElement[]数组，里面包含了类名，方法名，字段名，行数信息。默认最多取3层,也就是StackTraceElemen数组的前3条。

通过栈中调用org.apache.flink.runtime.io.network.buffer.LocalBufferPool.requestBufferBuilderBlocking
的比例来表示背压级别。requestBufferBuilderBlocking方法其实是申请buffer的方法，后面具体介绍。

ratio = (类名+方法名是requestBufferBuilderBlocking的StackTraceElement) / StackTraceElement[]的size

ratio小于0.1为OK,小于0.5为LOW, 大于等于0.5为High