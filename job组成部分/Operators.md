# Operators

## 1，windows

- window分为keyed window和non-keyed window，区别就是是否通过key分区，一个使用keyBy()和window(),一个使用windowAll()
- 不管是否是keyed window，window()和windowAll()方法都需提供window assigner,用来确定数据流应该被分配到哪一个或多个window中。

Flink 自带window如下：

### 1.1 Tumbling windows

- 滚动窗口，互不重叠。

### 1.2 Sliding windows

- 滑动窗口分配器可设置窗口大小，配置参数slide如果小于窗口大小，则窗口会有重叠，元素会属于多个窗口。

### 1.3 session windows

- 会话窗口不重叠并且没有固定的开始和结束时间。 当会话窗口在一段时间内没有接收到元素时，即当发生不活动的间隙时，会话窗口关闭。 会话窗口分配器可以配置静态会话间隙或会话间隙提取器功能，该功能定义不活动时间段的长度。 当此期限到期时，当前会话将关闭，后续元素将分配给新的会话窗口。
- 在内部，会话窗口operator为每个到达的记录创建一个新窗口，如果它们彼此之间的距离比定义的间隙更接近，则将窗口合并在一起。 为了可合并，会话窗口operator需要合并触发器和合并窗口函数，例如ReduceFunction，AggregateFunction或ProcessWindowFunction（FoldFunction无法合并）

### 1.4 Global windows

- 全局窗口分配器将具有相同键的所有元素分配给同一个全局窗口。 此窗口方案仅在指定了自定义触发器时才有用。 否则，将不执行任何计算，因为全局窗口没有我们可以处理聚合元素的自然结束。