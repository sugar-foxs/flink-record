消息是：UpdateTaskExecutionState(taskExecutionState)
taskmanager接收到task传来的状态信息，并转发给jobmanager
jobmanager更新task状态，更新的结果返回给taskmanager，由taskmanager来标记task为失败状态。
jobmanager更新task状态过程：
1,running:
-> CAS 修改task状态从部署中到运行中；
-> 修改成功，
-> 修改失败，
2,finished:
3,canceled:
4,failed: