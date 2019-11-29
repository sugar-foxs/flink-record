# State and Fault Tolerance

有状态的function和operator存储数据作为状态，以实现更复杂操作。

## 1，state分类

这一节说明有哪些种类state

- Keyed State

Keyed state只能在KeyedStream的function和operator中使用。可以将Keyed State视为已分区或sharded，每个key只有一个状态分区。每个keyed state在逻辑上绑定到<parallel-operator-instance，key>的唯一复合，并且由于每个键只属于一个并行实例，我们可以将其简单地视为<operator，key >。

Keyed state进一步组织成key group，是Flink可以重新分配key state的原子单元;key group数量与定义的最大并行度完全一样多。在执行期间，keyed operator的每个并行实例都使用一个或多个group的key。

- Operator state

使用operator 状态（或非keyed state），每个operator state都绑定到一个并行operator实例。 Kafka Connector是在Flink中使用operator state的一个很好的范例。 Kafka consumer的每个并行实例都将topic-partition和offset形成的map作为Operator状态存储。

Operator state接口支持在并行度更改时，在并行operator实例之间重新分配状态。可以有不同的方案进行此重新分配。

### 2，state存储形式

state有两种存储形式：managed ,raw.

managed状态由Flink运行时控制的数据结构来表示，例如内部哈希表或RocksDB。例如“ValueState”，“ListState”等.Flink运行时对状态进行编码并将它们写入检查点。

raw状态是operators保留在自己的数据结构中的状态。当启用检查点时，它们只会将一个字节序列写入检查点。 Flink对状态的数据结构一无所知，只看到原始字节。

所有数据流功能都可以使用managed状态，但raw状态接口只能在实现operator时使用。建议使用managed状态（而不是raw状态），因为在mamanged状态下，Flink能够在并行度更改时自动重新分配状态，并且还可以进行更好的内存管理。

### 3，Managed keyed state

- 这种state智能使用在keyedStream之后，这种state具有的类型有：
  - valueState
  - ListState，存储一个list
  - ReducingState，存储了所有添加到state的聚合，内部使用的是ReduceFunction
  - AggregatingState，添加到state的值的聚合，不过输出类型可能与输入不一样，内部使用的是AggregateFunction
  - FoldingState，添加到state的值的聚合，不过输出类型可能与输入不一样，内部使用的是FoldFunction,已被弃用
  - MapState，存储了一个map
- state只能使用RuntimeContext获取，所以只能在RichFunction中
- StateTtlConfig控制state的存活时间，现在过期的state默认是不会删除的，在被read之后会删除。

### 4，Managed operator state

- 实现CheckpointedFunction接口来使用非key state
- initializeState方法包含了状态初始化逻辑和恢复逻辑。
- snapshotState方法用来存储checkpoint状态
- operator state可以是分布在并行tasks中，或者在每个并行tasks中都有一份全量state.

### 5，Broadcast State

示例：一个流中包含一组规则，针对另一个流中的所有元素进行评估。

这需要将规则进行广播。

- 广播状态保存在内存中
- 所有的task保存一份广播状态

### 6，State Schema Evolution

目前，chema演变只支持Avro,

