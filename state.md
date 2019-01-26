# State
刚开始不知道state是做什么用的，现在网上搜了下，说：Flink通过异步的checkpoint机制来实现流式处理过程中的容错，简单来讲就是定时地将本地的状态序列化到一个持久存储中，当出现错误时通过恢复检查点的状态来实现容错。
下面就去看看state可以存什么类型数据和可以存在哪里。
StateBackend决定了state怎么存和存在哪里。
1，MemoryStateBackend
2，FsStateBackend
The FsStateBackend is encouraged for:
	1，Jobs with large state, long windows, large key/value states.
	2，All high-availability setups.
3，RocksDBStateBackend
The RocksDBStateBackend is encouraged for:
	1，Jobs with very large state, long windows, large key/value states.
	2，All high-availability setups.
RocksDBStateBackend is currently the only backend that offers incremental checkpoints.

#basic state 类型
Keyed State 和 Operator State




# queryable state
