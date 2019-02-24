# Table and SQL

## 1, Dynamic Table & Continuous Queries

steam和dynamic table、continuous queries的关系：

- 流被转换为动态表。

- 对动态表进行连续查询，生成新的动态表。

- 生成的动态表被转换回流。

## 2, Temporal Table Function

- 1.7增加了temporal table function,现在只可以在join中使用

- 定义temporal table function,示例

- ```java
  TemporalTableFunction rates = ratesHistory.createTemporalTableFunction("r_proctime", "r_currency");
  tEnv.registerFunction("Rates", rates);
  ```

## 3, Idle State Retention Time

设置State 保留时间是为了防止查询state过大。

## 4, Connect to External System

- connector描述了存储表数据的外部系统。  connector可能已经提供了带有字段的固定格式和schema。
- 有些系统支持不同的数据格式。 例如，存储在Kafka或文件中的表可以使用CSV，JSON或Avro对rows进行编码。 数据库连接器可能需要此处的表模式。 每个connector都记录了存储系统是否需要定义格式。 不同的系统还需要不同类型的格式（例如，面向列的格式与面向行的格式）。
- table schema定义了向SQL queries公开的表的模式。 它描述了源如何将数据格式映射到表模式，反之亦然。 schema可以访问连接器或格式定义的字段。 它可以使用一个或多个字段来提取或插入时间属性。 如果输入字段没有确定的字段顺序，则schema需要清晰地定义列名称，顺序和来历。
- 为了控制表的事件时间行为，Flink提供了预定义的时间戳提取器和水印策略。
- UpdateMode
  - 对于流式查询，需要声明如何在动态表和外部连接器之间执行转换。 更新模式指定connector应与外部系统交换哪种消息。
  - Append Mode：只交换INSERT消息。
  - Retract Mode：交换ADD和Retract消息。INSERT被编码为ADD消息，DELETE被编码为RETRACT消息，UPDATE被编码为更新（前一个）行的RETRACT消息和更新（新）行的ADD消息。 在此模式下，不能定义键。 但是，每次更新都包含两个效率较低的消息。
  - Upsert Mode：交换UPSERT和DELETE消息。 此模式需要一个（可能是复合的）唯一键，通过该键可以传播更新。 connector需要知道唯一键属性才能正确应用消息。 INSERT和UPDATE被编码为UPSERT消息。 DELETE为DELETE消息。 与retract stream的主要区别在于UPDATE更改使用单个消息进行编码，因此更有效。

- Connectors
  - File System Connector:source和sink都支持batch、streaming append mode，Format只支持csv。-
  - Kafka Connector:source和sink支持streaming append mode，读写format保持一致。
  - Elasticsearch Connector：sink支持streaming append mode 和streaming upsert mode。format只支持json。





