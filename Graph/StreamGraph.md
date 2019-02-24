#  StreamGraph

### 1，在执行execute之前，代码中DataStream会进行一系列转化，比如map:
```
public <R> SingleOutputStreamOperator<R> map(MapFunction<T, R> mapper) {
        //获取map输出类型信息
		TypeInformation<R> outType = TypeExtractor.getMapReturnTypes(clean(mapper), getType(),
				Utils.getCallLocationName(), true);

		return transform("Map", outType, new StreamMap<>(clean(mapper)));
	}
```
outTypeInfo是下一个operator的输出类型信息，每一种转换都会调用transform方法：
```
public <R> SingleOutputStreamOperator<R> transform(String operatorName, TypeInformation<R> outTypeInfo, OneInputStreamOperator<T, R> operator) {

		// read the output type of the input Transform to coax out errors about MissingTypeInfo
		transformation.getOutputType();

        //构造下一个operator的transformation
		OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
				this.transformation,
				operatorName,
				operator,
				outTypeInfo,
				environment.getParallelism());

		@SuppressWarnings({ "unchecked", "rawtypes" })
		SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, resultTransform);
        //将生成的transformation加入到env的list中
		getExecutionEnvironment().addOperator(resultTransform);

		return returnStream;
	}
```
就是说trasform方法生成的每一个operator都有自己的transformation，且transformation中包含了前一个transformation的信息，这样便形成了transformation的链，并会将每一个transformation加到env的list中，在执行execute时，构造StreamGraph.

### 2，通过transformations构造StreamGraph的过程如下：
```
private StreamGraph generateInternal(List<StreamTransformation<?>> transformations) {
		for (StreamTransformation<?> transformation: transformations) {
			transform(transformation);
		}
        // 到这，StreamGraph已经形成
		// 下面是为每个StreamNode配置资源
		。。。

		return streamGraph;
	}
```
重点是transform方法：
```
private Collection<Integer> transform(StreamTransformation<?> transform) {
        // 记录已经转换过的transformation,防止在循环里出不来
		if (alreadyTransformed.containsKey(transform)) {
			return alreadyTransformed.get(transform);
		}

		。。。

		// call at least once to trigger exceptions about MissingTypeInfo
		transform.getOutputType();

        //转换每种类型的transformation
		Collection<Integer> transformedIds;
		if (transform instanceof OneInputTransformation<?, ?>) {
			transformedIds = transformOneInputTransform((OneInputTransformation<?, ?>) transform);
		} else if (transform instanceof TwoInputTransformation<?, ?, ?>) {
			transformedIds = transformTwoInputTransform((TwoInputTransformation<?, ?, ?>) transform);
		} else ...

		// need this check because the iterate transformation adds itself before
		// transforming the feedback edges
		if (!alreadyTransformed.containsKey(transform)) {
			alreadyTransformed.put(transform, transformedIds);
		}

		。。。省略设置属性

		return transformedIds;
	}
```
以transformOneInputTransform为例，看看具体转换过程：
```
private <IN, OUT> Collection<Integer> transformOneInputTransform(OneInputTransformation<IN, OUT> transform) {
        // 先执行前一个transformation的转换，返回transformation的id,因为是OneInputTransformation，所以id只有一个
		Collection<Integer> inputIds = transform(transform.getInput());

		// the recursive call might have already transformed this
		if (alreadyTransformed.containsKey(transform)) {
			return alreadyTransformed.get(transform);
		}

		String slotSharingGroup = determineSlotSharingGroup(transform.getSlotSharingGroup(), inputIds, false);

		streamGraph.addOperator(transform.getId(),
				slotSharingGroup,
				transform.getOperator(),
				transform.getInputType(),
				transform.getOutputType(),
				transform.getName());

		if (transform.getStateKeySelector() != null) {
			TypeSerializer<?> keySerializer = transform.getStateKeyType().createSerializer(context.getExecutionConfig());
			streamGraph.setOneInputStateKey(transform.getId(), transform.getStateKeySelector(), keySerializer);
		}

		streamGraph.setParallelism(transform.getId(), transform.getParallelism());
		streamGraph.setMaxParallelism(transform.getId(), transform.getMaxParallelism());
		streamGraph.setMainOutputDamBehavior(transform.getId(), transform.getDamBehavior());
        // 为每个input和自己建立边
		for (Integer inputId: inputIds) {
			streamGraph.addEdge(inputId, transform.getId(), 0);
		}

		return Collections.singleton(transform.getId());
	}
```
在transform每一个streamTransformation的时候，都会生成一个streamNode，通过streamTransformation的前后关系生成前后streamNode之间的边streamEdge，并把streamEdge加入到前一个streamNode的输出边列表中和后一个streamNode的输入边列表中。通过递归调用遍历完所有streamTransformation，streamGraph便生成了。