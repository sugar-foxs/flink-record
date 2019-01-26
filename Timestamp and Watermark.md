# Timestamp and Watermark

- timestamp assigner用来产生timestamp和watermark, 有些 stream source可以自己产生timestamp到记录中和产生水印，这些source不需要timestamp assigner,但如果timestamp assigner被启用，则会覆盖source中的timestamp和水印。

- timestamp assigner定义的地方，一般是在解析map和过滤filter之后，任何情况下一定要在第一个使用事件时间操作之前（比如第一个window操作），特殊情况下，可以在kafka source内部。

## 实现Assigner

可通过实现下面的接口来实现assigner

### 1,AssignerWithPeriodicWatermarks

周期性的产生水印，水印可能依赖于流，也可能只基于processing time.

- 前提元素乱序，但是假定某个时间戳t的最新元素将在时间戳t的最早元素之后的最多n毫秒处到达。

- 就是说，现在是时间t,watermark之前的所有数据是假设都已经到达了。（watermark一定小于时间t）

- t-watermark是最大允许乱序时间，这个值根据不同的实现可能是固定的，也可能是跟事件时间相关并动态变化的。

### 2，AssignerWithPunctuatedWatermarks

带断点的水印，无论何时一个特定的事件表明一个新的watermark可能需要被创建，都使用AssignerWithPunctuatedWatermarks来生成。



## Flink自带的timestamp分配器

### 1，AscendingTimestampExtractor

- 定期水印生成的最简单的特殊情况是源任务看到的时间戳按升序发生的情况。在这种情况下，当前时间戳始终可以充当水印，因为没有更早的时间戳会到达。

### 2，BoundedOutOfOrdernessTimestampExtractor

- 水印滞后于在流中看到的最大（事件时间）时间戳一段固定的时间。这种情况包括预先知道流中可能遇到的最大延迟的情况，例如，在创建包含时间戳的元素的自定义源时，这些元素在固定的时间段内传播以进行测试。

### 3，IngestionTimeExtractor

根据机器的wall clock 分配时间。

对于AssignerWithPunctuatedWatermarks，flink没有自己实现，因为这个需要根据业务情况。



# waterMark的产生和传输

在timeCharacteristic是EventTime和IngestionTime时需要watermark。下面介绍一下EventTime的情况：

传输方式是streamSource使用output向后传输watermark，然后每一个operator向后传输。

watermark值如何变化？watermark是record中的timeStamp

