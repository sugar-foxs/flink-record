ExecutionGraph.scheduleForExecution()是真正执行的开始。
1，使用CAS尝试改变job状态从已创建到运行中。如果修改成功，执行第二步。
2，执行调度。调度模式有两种：懒调度，task在输入数据到达的时候再启动；立即调度，所有task即刻调度。默认是立即调度。
参数都包含一个SlotProvider，用来为任务分配slot.

3,allocateAndAssignSlotForExecution负责分配资源
    -> CAS改变task状态从已创建到已调度
    -> 计算出最合适的位置
    -> slotProvider.allocateSlot  [scheduler]
        -> task有slot共享组，用共享的slots调度
            -> 从组里取slotFromGroup，分配new slot并放到组里。如果new slot为null,使用slotFromGroup；如果new slot不为null，slotFromGroup为null,或者new slot在本地，使用new slot；否则使用slotFromGroup。
        -> 不带共享slot的调度
            -> 这个比较简单，getFreeSlotForTask获取slot，如果没取到，则把task放到等待队列中。在收到有空闲slot的通知后，再为队列中的task分配slot。
            -> Instance代表一个TaskManager.注册了一个新的taskManager,便会通知scheduler
4,deploy,部署
    -> CAS改变task状态从已创建或者已调度到部署中。
    -> 创建TaskDeploymentDescriptor deployment，task的信息包含其中。
    -> taskManagerGateway.submitTask（deployment）向taskmanager提交任务。