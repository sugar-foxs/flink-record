# latency （flink1.3）
这篇主要介绍一下flink1.3中metrics中的延时指标latency,下面通过代码回溯分析latency：

## 1，latency存储
从源码首先可以找到latency是存储在LatencyGauge中的，抽象类AbstractStreamOperator是所有operator的基类，所以每个operator中都存储了一份（提示：在之后的版本中latency已经不存储在gauge中了，现在存储在Histogram中，详情看高版本源码）。
		
	public abstract class AbstractStreamOperator<OUT>                   （1）
	implements StreamOperator<OUT>, Serializable, KeyContext {
		...			
		protected transient LatencyGauge latencyGauge;
        ...
		protected void reportOrForwardLatencyMarker(LatencyMarker marker) {
		// all operators are tracking latencies
		this.latencyGauge.reportLatency(marker, false);

		// everything except sinks forwards latency markers
		this.output.emitLatencyMarker(marker);
		}
		
		protected static class LatencyGauge implements Gauge<Map<String, HashMap<String, Double>>> {
			private final Map<LatencySourceDescriptor, DescriptiveStatistics> latencyStats = new HashMap<>();
			private final int historySize;
	
			LatencyGauge(int historySize) {
				this.historySize = historySize;
			}
	        //这就是将latency存储在operator中的方法
			public void reportLatency(LatencyMarker marker, boolean isSink) {
				LatencySourceDescriptor sourceDescriptor = LatencySourceDescriptor.of(marker, !isSink);
				DescriptiveStatistics sourceStats = latencyStats.get(sourceDescriptor);
				if (sourceStats == null) {
					sourceStats = new DescriptiveStatistics(this.historySize);
					latencyStats.put(sourceDescriptor, sourceStats);
				}
				long now = System.currentTimeMillis();
                
				sourceStats.addValue(now - marker.getMarkedTime());
			}
			...
		}

	}
从LatencyGauge代码中可以看出，latency存储在HashMap中，key为LatencySourceDescriptor， Value为DescriptiveStatistics，那么key和value到底存了什么：

	private static class LatencySourceDescriptor {                     （2）
	
		//A unique ID identifying a logical source in Flink.
		private final int vertexID;

	 	//Identifier for parallel subtasks of a logical source.
		private final int subtaskIndex;
		
		public static LatencySourceDescriptor of(LatencyMarker marker, boolean ignoreSubtaskIndex) {
			if (ignoreSubtaskIndex) {
				//operator不是streamSink时，忽略subtask index，统一用-1标识
				return new LatencySourceDescriptor(marker.getVertexID(), -1);
			} else {
				//operator是streamSink时
				return new LatencySourceDescriptor(marker.getVertexID(), marker.getSubtaskIndex());
			}

		}
		......
	}

	public class DescriptiveStatistics implements StatisticalSummary, Serializable {
		......
	    //eDA中存储了latency数据
	    private ResizableDoubleArray eDA = new ResizableDoubleArray();
        ......
	}

从代码中可见LatencySourceDescriptor中存储了source的唯一id和并行子任务的标识符，DescriptiveStatistics中使用ResizableDoubleArray储存latency数据。

    public class ResizableDoubleArray implements DoubleArray, Serializable {（3）
	    ...
		private static final int DEFAULT_INITIAL_CAPACITY = 16;
	    private double[] internalArray;
		...
	}
ResizableDoubleArray中实际上是一个double类型的数组，初始容量是16，数组的一系列操作第四节再讨论，现在只要知道这个数组里存着一定数量的latency，通过这个latency数组便可以得到一段时间内的latency的均值，最大值，最小值和p50,p99等分位数值。（ps:p50指小于这个值的数据占总数量的50%）

见下方SreamSink代码，只有StreamSink重写了AbstractStreamOperator的方法，isSink为true,其他的operator都是使用AbstractStreamOperator的方法，见代码段（1）reportOrForwardLatencyMarker方法，isSink为false。

	public class StreamSink<IN> extends AbstractUdfStreamOperator<Object, SinkFunction<IN>>
		implements OneInputStreamOperator<IN, Object> {
	    ...
		@Override
		protected void reportOrForwardLatencyMarker(LatencyMarker maker) {
			// all operators are tracking latencies
			this.latencyGauge.reportLatency(maker, true);
	
			// sinks don't forward latency markers
		}
	}


**总结**

- latency存储在key为LatencySourceDescriptor，vlaue为DescriptiveStatistics的map中，
- LatencySourceDescriptor中存储了source的唯一id和并行子任务的标识符，只有operator是StreamSink时才会记录并行子任务的标识符，否则都用-1代替。
- DescriptiveStatistics用数组储存了latency数据。

**但是latency代表什么实际意义呢？**

从代码段（1）的reportLatency方法中

	sourceStats.addValue(now - marker.getMarkedTime());                  （4）

这条语句可以知道，latency是现在时刻减去LatencyMarker中存储的时间戳的值，所以只要找到LatencyMarker是什么时间生成的，就知道了latency代表什么意思，看下一节。

## 2，LatencyMarker如何形成的
通过一层层回溯，到LatencyMarksEmitter中

	public LatencyMarksEmitter(                                         （5）
			final ProcessingTimeService processingTimeService,
			final Output<StreamRecord<OUT>> output,
			long latencyTrackingInterval,
			final OperatorID operatorId,
			final int subtaskIndex) {
		latencyMarkTimer = processingTimeService.scheduleAtFixedRate(
			new ProcessingTimeCallback() {
			@Override
			public void onProcessingTime(long timestamp) throws Exception {
				try {
					// ProcessingTimeService callbacks are executed under the checkpointing lock
					output.emitLatencyMarker(new LatencyMarker(timestamp, operatorId, subtaskIndex));
				} catch (Throwable t) {
					...
				}
			}
		},
		0L,
		latencyTrackingInterval);
	}

可见主要方法是processingTimeService.scheduleAtFixedRate，周期产生latencyMarker(携带其产生时的时间戳)，latencyTrackingInterval默认2000毫秒。

	public ScheduledFuture<?> scheduleAtFixedRate(ProcessingTimeCallback callback, long initialDelay, long period) {                      
        //时间是现在的时间加初始延迟时间
		long nextTimestamp = getCurrentProcessingTime() + initialDelay;（6）

		try {
			return timerService.scheduleAtFixedRate(
				new RepeatedTriggerTask(status, task, checkpointLock, callback, nextTimestamp, period),
				initialDelay,
				period,
				TimeUnit.MILLISECONDS);
		} catch (RejectedExecutionException e) {
			...
		}
	}

	public long getCurrentProcessingTime() {
		return System.currentTimeMillis();
	}

继续追踪RepeatedTriggerTask，其run方法如下：

	public void run() {                                              （7）
		synchronized (lock) {
			try {
				if (serviceStatus.get() == STATUS_ALIVE) {
                    //这个方法就是代码段（5）ProcessingTimeCallback的方法
					target.onProcessingTime(nextTimestamp);
				}
				nextTimestamp += period;
			} catch (Throwable t) {
				...
			}
		}
	}

所以其实是周期执行下面这句，每次生成一个LatencyMarker，timestamp一开始是当时时间戳加初始延迟时间，之后是每次加一个period(就是latencyTrackingInterval，这个可以在配置文件中配置)。

	output.emitLatencyMarker(new LatencyMarker(timestamp, operatorId,  subtaskIndex));（8）

到这，肯定还有一些疑问，到底是什么时候调用这个方法的，我们继续回溯，可以看到是在StreamSource类的run方法中新建了LatencyMarksEmitter：

	public void run(final Object lockingObject,
			final StreamStatusMaintainer streamStatusMaintainer,
			final Output<StreamRecord<OUT>> collector) throws Exception {

		final TimeCharacteristic timeCharacteristic = getOperatorConfig().getTimeCharacteristic();

		LatencyMarksEmitter latencyEmitter = null;
		if (getExecutionConfig().isLatencyTrackingEnabled()) {
			//创建LatencyMarksEmitter
			latencyEmitter = new LatencyMarksEmitter<>(
				getProcessingTimeService(),
				collector,
				getExecutionConfig().getLatencyTrackingInterval(),
				getOperatorConfig().getVertexID(),
				getRuntimeContext().getIndexOfThisSubtask());
		}

		final long watermarkInterval = getRuntimeContext().getExecutionConfig().getAutoWatermarkInterval();

		this.ctx = StreamSourceContexts.getSourceContext(
			timeCharacteristic,
			getProcessingTimeService(),
			lockingObject,
			streamStatusMaintainer,
			collector,
			watermarkInterval,
			-1);

		try {
			userFunction.run(ctx);

			// if we get here, then the user function either exited after being done (finite source)
			// or the function was canceled or stopped. For the finite source case, we should emit
			// a final watermark that indicates that we reached the end of event-time
			if (!isCanceledOrStopped()) {
				ctx.emitWatermark(Watermark.MAX_WATERMARK);
			}
		} finally {
			...
		}
	}

	
所以latencyMarker是StreamSource类型的operator（即job的第一个执行的operator）定期产生的,并将lactencyMarker按顺序传递给之后的每一个operator，怎么传递看下一节。

**总结**：

- LatencyMarker是job的第一个执行的operator定期生成的，所以每个operator中的latency代表latencyMarker从由StreamSource生成至到达当前operator花费的时间，这样对于需要处理的数据，则可近似理解为数据处理的延时时间。

## 3.LatencyMarker如何传递的

代码段（5）中emitLatencyMarker是传递LatencyMarker的方法，根据Output的类型不同，emitLatencyMarker有不同的实现方式，每个opertor中都包含一个Output，一种是ChainingOutput，一种是RecordWriterOutput。

ChainingOutput中包含chain中下一个operator的引用，负责operator chain中传输数据，最终调用的是这个方法：

	protected void reportOrForwardLatencyMarker(LatencyMarker marker) {  （10）
		//记录latency		
		this.latencyGauge.reportLatency(marker, false);
        //传递给下一个
		this.output.emitLatencyMarker(marker);
	}

RecordWriterOutput负责subtask之间传递：

	public class RecordWriterOutput<OUT> implements Output<StreamRecord<OUT>> {

		private StreamRecordWriter<SerializationDelegate<StreamElement>> recordWriter;
	
		private SerializationDelegate<StreamElement> serializationDelegate;
	
		private final StreamStatusProvider streamStatusProvider;
	
		private final OutputTag outputTag;
		...
		@Override
		public void emitLatencyMarker(LatencyMarker latencyMarker) {
			serializationDelegate.setInstance(latencyMarker);
	
			try {
				recordWriter.randomEmit(serializationDelegate);
			}
			catch (Exception e) {
				throw new RuntimeException(e.getMessage(), e);
			}
		}
		...
	}

一直追溯下去，可以看到把数据通过netty传输到了对应的channel，这个涉及到task之间是如何传递数据的，以后具体分析。
以OneInputStreamTask为例，看subtask接收到数据如何处理的。

	public class OneInputStreamTask<IN, OUT> extends StreamTask<OUT, OneInputStreamOperator<IN, OUT>> {                

		private StreamInputProcessor<IN> inputProcessor;                  （11）
	
		@Override
		protected void run() throws Exception {
			// cache processor reference on the stack, to make the code more JIT friendly
			final StreamInputProcessor<IN> inputProcessor = this.inputProcessor;
	
			while (running && inputProcessor.processInput()) {
				// all the work happens in the "processInput" method
			}
		}
	}

进入processInput()方法，省略了不相干代码：

	public boolean processInput() throws Exception {                     （12）
		while (true) {
			if (currentRecordDeserializer != null) {
				if (result.isFullRecord()) {
					if (recordOrMark.isWatermark()) {
						...
					} else if (recordOrMark.isStreamStatus()) {
						...
					} else if (recordOrMark.isLatencyMarker()) {
						// 处理latencyMarker
						synchronized (lock) {
							streamOperator.processLatencyMarker(recordOrMark.asLatencyMarker());
						}
						continue;
					} else {
						return true;
					}
				}
			}
		}
	}

- processLatencyMarker调用的就是上面代码段（10）reportOrForwardLatencyMarker方法，可以看出，既会记录latency,也会转发LatencyMarker到下一个operator。
- 因为ChainingOutput和subtask的第一个operator都会记录latency，这样一来，其实所有的operator都会记录latency。

**总结**：

- latency表示LatencyMarker从StreamSource的headOperator产生到各个operator所用的时间,是不包含用户数据处理逻辑花的时间的，所以latency便可以代表数据处理延时。

## 4.latency存储到数组里，数组是如何变化的
存储latency的真正逻辑是这句：

	sourceStats.addValue(now - marker.getMarkedTime());

进去看看：

	public void addValue(double v) {
	    if (windowSize != INFINITE_WINDOW) {
	        if (getN() == windowSize) {
                //如果元素个数等于窗口大小
	            eDA.addElementRolling(v);
	        } else if (getN() < windowSize) {
	            eDA.addElement(v);
	        }
	    } else {
	        eDA.addElement(v);
	    }
    }

窗口大小默认128，因为从代码块（1）中

	sourceStats = new DescriptiveStatistics(this.historySize);

和DescriptiveStatistics的构造函数
   
	public DescriptiveStatistics(int window) throws MathIllegalArgumentException {
        setWindowSize(window);
    }

可以看出historySize是DescriptiveStatistics的windowSize，historySize可配置，默认128

	public static final ConfigOption<Integer> LATENCY_HISTORY_SIZE =
		key("metrics.latency.history-size")
		.defaultValue(128);

下面看下addElementRolling和addElement函数的逻辑
### 4.1 addElement，元素个数小于窗口大小或者窗口大小等于-1时调用
	
	public synchronized void addElement(double value) {
		//数组长度小于等于开始下标加元素个数
        if (internalArray.length <= startIndex + numElements) {
            expand();
        }
		//新数据加在数组元素后面
        internalArray[startIndex + numElements++] = value;
    }

进入expand()方法

	protected synchronized void expand() {
        //expansionFactor默认是2.0
        int newSize = 0;
        if (expansionMode == ExpansionMode.MULTIPLICATIVE) {
            //扩容模式是乘上expansionFactor
			newSize = (int) FastMath.ceil(internalArray.length * expansionFactor);
        } else {
			//扩容模式是加上expansionFactor
            newSize = (int) (internalArray.length + FastMath.round(expansionFactor));
        }
        final double[] tempArray = new double[newSize];
        // 将元素复制到新数组中
        System.arraycopy(internalArray, 0, tempArray, 0, internalArray.length);
        internalArray = tempArray;
    }

所以元素个数小于窗口大小或者窗口大小等于-1时元素添加到最后，数组满了便扩容，直到元素个数等于窗口大小，调用addElementRolling。

### 4.2 addElementRolling，元素个数等于窗口大小时调用

	public synchronized double addElementRolling(double value) {
        double discarded = internalArray[startIndex];

        if ((startIndex + (numElements + 1)) > internalArray.length) {
            expand();
        }
        // start index后移一位
        startIndex += 1;
        // 新元素存储在扩容后旧元素后的空位
        internalArray[startIndex + (numElements - 1)] = value;
        // 检查是否需要收缩
        if (shouldContract()) {
            contract();
        }
		//startIndex之前的数据是伪删除，维持有效数据个数是窗口大小
        return discarded;
    }

因为一直扩容肯定是不行的，看看什么时候会收缩

	private synchronized boolean shouldContract() {
        if (expansionMode == ExpansionMode.MULTIPLICATIVE) {
            return (internalArray.length / ((float) numElements)) > contractionCriterion;
        } else {
            return (internalArray.length - numElements) > contractionCriterion;
        }
    }

contractionCriterion默认等于2.5，所以

- 扩容模式是乘积时，数组长度是元素个数的2.5倍时收缩。
- 扩容模式是加法时，数组长度比元素个数大2.5时收缩。

看下如何收缩的：

		public synchronized void contract() {
	        final double[] tempArray = new double[numElements + 1];
	
	        // 新数组容量是元素个数加1，将有效数据复制到新数组中，变相删除了窗口外的数据
	        System.arraycopy(internalArray, startIndex, tempArray, 0, numElements);
	        internalArray = tempArray;
	
	        // 重置start index为0
	        startIndex = 0;
	    }

所以收缩策略是新建容量是窗口大小加1的新数组，将有效数据复制到新数组中，则变相删除了窗口外的数据。

**总结**：

- 数组元素个数未达到窗口大小时，元素正常插入到最后；达到窗口大小时，维持最新的有效数据的数量一直是窗口大小，通过扩容和收缩策略来删除无效数据，保证数组不会过大。