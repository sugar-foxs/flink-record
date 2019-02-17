## JobManager

- jobManager 负责接收flink jobs，任务调度，收集job状态，和管理taskManager

  继承自FlinkActor，它主要接收以下信息：
  - RegisterTaskManager: 它由想要注册到JobManager的TaskManager发送。注册成功会通过AcknowledgeRegistration消息进行Ack。
  - SubmitJob: 由提交作业到系统的Client发送。提交的信息是JobGraph形式的作业描述信息，所有flink作业都会抽象成JobGraph交给JobManager处理。
  - CancelJob: 请求取消指定id的作业。成功会返回CancellationSuccess，否则返回CancellationFailure。
  - UpdateTaskExecutionState: 由TaskManager发送，用来更新执行节点(ExecutionVertex)的状态。成功则返回true，否则返回false。
  - RequestNextInputSplit: TaskManager上的Task请求下一个输入split，成功则返回NextInputSplit，否则返回null。
  - JobStatusChanged： 它意味着作业的状态(RUNNING, CANCELING, FINISHED,等)发生变化。这个消息由ExecutionGraph通知。
