有一个需求，需要在窗口内定时触发窗口的聚合计算，实现自定义窗口触发器通过继承Trigger，并传入触发时间间隔interval作为参数。

```
class TimingTrigger(interval: Long) extends Trigger[CRow,TimeWindow] with Logging
```

Trigger中有4个方法需要覆盖，onElement(),onProcessingTime(),onEventTime(),clear()。

onElement：每个元素进入窗口都会被调用。

onProcessingTime：当注册的处理时间计时器启动时触发。

onEventTime：当注册的事件事件计时器启动时触发。

clear：执行清除逻辑。

上面三个通过返回TriggerResult来决定之后进行什么操作。

TriggerResult.CONTINUE : 什么都不做

TriggerResult.FIRE : 触发计算

TriggerResult.PURGE ： 清除窗口数据

TriggerResult.FIRE_AND_PURGE ：触发计算并清除窗口数据



```
class TimingTrigger(interval: Long) extends Trigger[CRow,TimeWindow] with Logging{

  //state用来存储触发时间
  val stateDesc = new ValueStateDescriptor[java.lang.Long](
    "triggerInterval", BasicTypeInfo.LONG_TYPE_INFO)

  //在onElement方法中注册窗口内最开始触发的任务
  override def onElement(
      element: CRow,
      timestamp: Long,
      window: TimeWindow,
      ctx: Trigger.TriggerContext): TriggerResult = {
    if (window.maxTimestamp <= ctx.getCurrentWatermark) {
      // if the watermark is already past the window fire immediately
      TriggerResult.FIRE
    } else {
      val fireTimestamp = ctx.getPartitionedState(stateDesc)
      if (fireTimestamp.value() == null && ctx.getCurrentWatermark > 0) {
        val start = Math.max(ctx.getCurrentWatermark, window.getStart)
        val nextFireTimestamp = start - start % interval + interval
        ctx.registerEventTimeTimer(nextFireTimestamp)
        fireTimestamp.update(nextFireTimestamp)
        return TriggerResult.CONTINUE
      }
      TriggerResult.CONTINUE
    }
  }

  override def onProcessingTime(
   time: Long,
   window: TimeWindow,
   ctx: Trigger.TriggerContext): TriggerResult = {
    TriggerResult.CONTINUE
  }
  
  //注册的定时任务触发时会回调这个方法
  override def onEventTime(
    time: Long,
    window: TimeWindow,
    ctx: Trigger.TriggerContext): TriggerResult = {
    if (window.maxTimestamp <= time) return TriggerResult.FIRE
    val fireTimestamp = ctx.getPartitionedState(stateDesc)
    if (fireTimestamp.value() != null && fireTimestamp.value() == time) {
      if (window.maxTimestamp <= ctx.getCurrentWatermark) {
        return TriggerResult.FIRE
      }
      //更新state中触发时间
      fireTimestamp.update(time + interval)
      //注册下一次定时任务
      ctx.registerEventTimeTimer(time + interval)
      //触发计算
      return TriggerResult.FIRE
    }
    TriggerResult.CONTINUE
  }

  override def clear(window: TimeWindow, ctx: Trigger.TriggerContext): Unit = {
    ctx.getPartitionedState(stateDesc).clear()
  }
}
```