# SQL


# User-defined Source & Sink
## TableSource
### BatchTableSource
### StreamTableSource
### a TableSource with Time Attributes
#### a Processing Time Attribute，通过实现DefinedProctimeAttribute接口
#### a Rowtime Attribute ，通过实现DefinedRowtimeAttributes 接口
### a TableSource with Projection Push-Down
#### TableSource通过实现ProjectableTableSource接口支持projection push-down。
#### 如果TableSource定义了一个具有嵌套模式的表，实现NestedFieldsProjectableTableSource接口。
###  a TableSource with Filter Push-Down，实现FilterableTableSource接口,可以先对数据进行过滤。

## TableSink
### BatchTableSink,批处理sink
### AppendStreamTableSink，存储只有insert操作的table
### RetractStreamTableSink，存储有insert, update, and delete操作的table
RetractStreamTableSink<T> extends TableSink<Tuple2<Boolean, T>> {//第一个字段true 表示 insert, false 表示 delete

  public TypeInformation<T> getRecordType();

  public void emitDataStream(DataStream<Tuple2<Boolean, T>> dataStream);
}
### UpsertStreamTableSink，该表必须具有唯一键字段或者是只能追加的表，
UpsertStreamTableSink<T> extends TableSink<Tuple2<Boolean, T>> {

  public void setKeyFields(String[] keys);//设置唯一键

  public void setIsAppendOnly(boolean isAppendOnly);//设置是否是只追加表。

  public TypeInformation<T> getRecordType();

  public void emitDataStream(DataStream<Tuple2<Boolean, T>> dataStream);
}

## TableFactory
TableFactory允许从基于字符串的属性创建不同的表相关的实例。
BatchTableSourceFactory: Creates a batch table source.
BatchTableSinkFactory: Creates a batch table sink.
StreamTableSoureFactory: Creates a stream table source.
StreamTableSinkFactory: Creates a stream table sink.
DeserializationSchemaFactory: Creates a deserialization schema format.
SerializationSchemaFactory: Creates a serialization schema format.

# User-defined function
大多数情况下，必须先注册用户自定义的function再在query中使用，在scala api中注册function不是必要的。
使用TableEnvironment 的registerFunction()注册，注册之后，会被插入到TableEnvironment的function目录，这样才会被Table API or SQL parser识别并翻译。
## ScalarFunction，标量函数
用户定义的标量函数将零个，一个或多个标量值映射到新的标量值。
public static class TimestampModifier extends ScalarFunction {
  public long eval(long t) {
    return t % 1000;
  }

  public TypeInformation<?> getResultType(signature: Class<?>[]) {
    return Types.TIMESTAMP;
  }
}
## TableFunction
用户定义的表函数将零个，一个或多个标量值作为输入参数。
public class CustomTypeSplit extends TableFunction<Row> {
    public void eval(String str) {
        for (String s : str.split(" ")) {
            Row row = new Row(2);
            row.setField(0, s);
            row.setField(1, s.length);
            collect(row);
        }
    }

    @Override
    public TypeInformation<Row> getResultType() {
        return Types.ROW(Types.STRING(), Types.INT());
    }
}
## AggregateFunction
public static class WeightedAvgAccum {
    public long sum = 0;
    public int count = 0;
}

/**
 * Weighted Average user-defined aggregate function.
 */
public static class WeightedAvg extends AggregateFunction<Long, WeightedAvgAccum> {

    @Override
    public WeightedAvgAccum createAccumulator() {
        return new WeightedAvgAccum();
    }

    @Override
    public Long getValue(WeightedAvgAccum acc) {
        if (acc.count == 0) {
            return null;
        } else {
            return acc.sum / acc.count;
        }
    }

    public void accumulate(WeightedAvgAccum acc, long iValue, int iWeight) {
        acc.sum += iValue * iWeight;
        acc.count += iWeight;
    }

    public void retract(WeightedAvgAccum acc, long iValue, int iWeight) {
        acc.sum -= iValue * iWeight;
        acc.count -= iWeight;
    }
    
    public void merge(WeightedAvgAccum acc, Iterable<WeightedAvgAccum> it) {
        Iterator<WeightedAvgAccum> iter = it.iterator();
        while (iter.hasNext()) {
            WeightedAvgAccum a = iter.next();
            acc.count += a.count;
            acc.sum += a.sum;
        }
    }
    
    public void resetAccumulator(WeightedAvgAccum acc) {
        acc.count = 0;
        acc.sum = 0L;
    }
}

// register function
StreamTableEnvironment tEnv = ...
tEnv.registerFunction("wAvg", new WeightedAvg());

// use function
tEnv.sqlQuery("SELECT user, wAvg(points, level) AS avgPoints FROM userScores GROUP BY user");













