# Event time

## Event time
![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/times_clocks.svg)
通过`env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);`来设置Time Characteristic。如果使用event time，需要使用直接定义event time且发送watermark的source或者在source之后定义`Timestamp Assigner & Watermark Generator`

### Event Time and Watermarks
Flink使用watermark来衡量event time的进度，watermark也是stream的一部分，携带时间戳t。当一个watermark到达operator的时候，这个operator可以将自己内部的event time clock的时间调整为watermark的时间。`Watermark(t)`表示stream中event time已经到达时间t，意味着stream中不应该再有时间小于t的数据。
![Ordered](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/stream_watermark_in_order.svg)

![out of order](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/stream_watermark_out_of_order.svg)

### Watermarks in Parallel Streams
watermark一般直接在source中直接产生或者在source之后产生，每个并行的source function的subtask通常都会独立产生自己的watermark，用来定义对应source的event time。当watermark流过stream，到达operator的时候，operator对应的event time也会推进，同时为后面的operator产生新的watermark。对于同时消费多个input的operator，其当前event time是所有input的event time的最小值。
![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/parallel_streams_watermarks.svg)

### Late Elements
时间小于watermark的元素，即时间戳小于event time clock的时间。


## Generating Timestamps / Watermarks
为了使用event time，Flink需要知道事件的时间戳，即stream中的每个元素都要有指定自己的event timestamp，通常是通过提取元素中的某些字段。有两种方式可以产生watermark:
- 直接在data stream source中
- 通过timestamp assigner / watermark generator，Flink中timestamp assigner也定义了要emit的watermark。这个会覆盖source中直接产生的timestamp

### Source Functions with Timestamps and Watermarks
要在source中直接产生timestamp，需要调用`SourceContext.collectWithTimestamp`，要产生watermark需要调用`emitWatermark(Watermark)`

### Timestamp Assigners / Watermark Generators
timestamp assigner通常在source的map和filter之后设置(使用`assignTimestampsAndWatermarks`方法)，必须在第一个event time的操作之前设置(比如第一个window操作)。Kafka比较特殊，可以在source(或者consumer)里面指定timestamp assigner/watermark emitter。

`AssignerWithPeriodicWatermarks`接口赋值timestamp，周期性调用`getCurrentWatermark`生成watermark。这个周期通过`ExecutionConfig.setAutoWatermarkInterval(...)`设置
```
public class BoundedOutOfOrdernessGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

```

`AssignerWithPunctuatedWatermarks`接口当事件满足产生watermark的条件时就产生一个watermark(一般事件是专门插入的特殊事件)，因此在调用`extractTimestamp`之后立刻调用`checkAndGetNextWatermark`看是否需要生成watermark。
```
public class PunctuatedAssigner implements AssignerWithPunctuatedWatermarks<MyEvent> {

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark checkAndGetNextWatermark(MyEvent lastElement, long extractedTimestamp) {
		return lastElement.hasWatermarkMarker() ? new Watermark(extractedTimestamp) : null;
	}
}
```

### Timestamps per Kafka Partition
每个kafka partition会有一个简单是event time模式(ascending timestamps or bounded out-of-orderness)，但是从kafka消费的时候通常是多个partition同时消费，破坏这种event time模式。这个时候可以使用`Kafka-partition-aware` watermark generation，watermark在kafka consumer中产生，每个partition产生一个，然后在类似shuffle中做合并(即取其中最小的一个).
![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/parallel_kafka_watermarks.svg)

## Pre-defined Timestamp Extractors / Watermark Emitters
`AscendingTimestampExtractor` 和 `BoundedOutOfOrdernessTimestampExtractor`


# State & Fault Tolerance

## Working with State
### Keyed State and Operator State
有两种基本的state:
- Keyed State，总是和key相关，只能用在一个`KeyStream`的function和operator上。可以看成是已经被分区的`Operator State`，每个key对应一个state-partition。逻辑上每个keyed-state对应一个唯一的组合`<parallel-operator-instance, key>`，由于每个key只属于唯一一个keyed operator的实例，可以简单认为是`<operator, key>`。keyed-state进一步被分组为`Key Groups`，是flink redistribute Keyed State的原子单位，其数量等于最大并行度。执行的时候每个keyed-operator的并行实例操作一个或者多个key group的key。
- Operator State，绑定到一个并行的operator实例，`Kafka Connector`是其中一个例子，其每个并行实例维护一份topic partition和offset的映射作为Operator State。当并行度发生变化的时候也支持在多个operator实例上重新分发状态。

### Raw and Managed State
Keyed State和Operator State都存在两种形式:
- `Managed State`有Flink runtime的数据结构控制，比如内部的hash map或者RocksDB，类似`ValueState`和`ListState`，Flink runtime会对其进行编码写入checkpoint
- `Raw State`是operator自己的数据结构保存的状态，checkpoint的时候只是写入二进制流，Flink对其结构一无所知

所有的datastream function可以使用Managed State，raw state只能用来实现operator。推荐使用managed state，flink运行时会对其做redistribute，内存管理也比较高效

### Using Managed Keyed State
可以使用是state primitive:
- ValueState<T>
- ListState<T>
- ReducingState<T>:
- AggregatingState<IN, OUT>
- MapState<UK, UV>

这些state对象只能用来和state交互，state可能存储在内存，也可以在磁盘或者其他地方。从state对象中获取到的值依赖于input元素的key。

为了获取state handle，需要创建一个`StateDescriptor`，包含state的名称、value的类型和用户指定的function。只能通过`RuntimeContext`访问state，因此只可能是通过[`rich functions`](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/api_concepts.html#rich-functions)。`RuntimeContext`有下面几个方法来获取状态:
```
ValueState<T> getState(ValueStateDescriptor<T>)
ReducingState<T> getReducingState(ReducingStateDescriptor<T>)
ListState<T> getListState(ListStateDescriptor<T>)
AggregatingState<IN, OUT> getAggregatingState(AggregatingState<IN, OUT>)
FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)
MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)
```

任何类型的keyed-state可以设置一个TTL，当TTL超时的时候会尽可能清除数据。
```
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
```
TTL也是存储在state里，因此会增加state的数据量；目前只支持processing-time。失效的数据不会自动清除，只有在被访问的时候才会清除掉。

### Using Managed Operator State
可以实现`CheckpointedFunction`或者`ListCheckpointed`，前者更加通用。需要实现两个方法:
```
void snapshotState(FunctionSnapshotContext context) throws Exception;
void initializeState(FunctionInitializationContext context) throws Exception;
```
当需要做checkpoint的时候会调用`snapshotState`；user-defined function初始化的时候会调用`initializeState`，可能是第一次初始化或者从前一次checkpoint恢复的时候。

目前支持list-style managed operator state，state是一个可序列化对象的列表，彼此互相独立，因此在rescale的时候方便redistribute，因此这些对象是redistribute的最小力度。有两种redistribution模式：
- `Even-split redistribution`，每个operator得到一个sublist，可能包含0个或者多个元素，逻辑上连接起来就是一个完整的state list。`restore/redistribution`的时候每个operator按并行度平分这个list
- `Union redistribution`，restore/redistribution`的时候每个operator得到整个list

## The Broadcast State Pattern
除了上面的even-split和union，还有第三种模式: `Broadcast State`。其使用场景是从一个stream进来的数据需要广播到所有downstream task，然后保存在内存中，用于处理其他stream进来的元素，比如一个stream包含rule元素，会用在其他stream的每个元素上面。

**`todo`**


## Checkpointing
用于保证state是fault tolerance，可以从stream的指定状态和位置恢复，达到failure-free的语义。做checkpoint有两个条件：
- 可以从指定时间replay的持久化存储，比如Kafka或者HDFS
- state的持久化存储，典型的是分布式存储，比如HDFS

checkpoint默认情况下是关闭的，通过调用`StreamExecutionEnvironment.enableCheckpointing(n)`启用，其中n是进行checkpoint的时间间隔。另外还有其他参数可以配置：
```
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000);

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// make sure 500 ms of progress happen between checkpoints
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig().setCheckpointTimeout(60000);

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// enable externalized checkpoints which are retained after job cancellation
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```

更多的参数可以通过`conf/flink-conf.yaml`进行配置

默认情况下state存储在TaskManager的内存中，checkpoint保存在JobManager的内存中，可以通过`StreamExecutionEnvironment.setStateBackend(…)`设置State Backend

目前不支持包含Iteration的job。

## Queryable State Beta
将managed keyed state暴露给外部，允许用户在flink之外查询状态。一些场景下减少分布式operation/transaction，另外对debug很有用。目前处于beta状态。

### Architecture
包含三部分：
- `QueryableStateClient`，运行在flink之外，提交查询
- `QueryableStateClientProxy`，运行在TaskManager中，接收客户端请求，返回给客户端
- `QueryableStateServer`，运行在TaskManager中，服务本地存储的状态

client连接到其中一个proxy，提交关于某个key的请求，由于keyed state按Key Group组织，每个TaskManager可能有多个Key Group，proxy会向JobManager询问对应key所在的TaskManager，然后请求其server，返回response给客户端。

### Activating Queryable State
功能默认是关闭，如果要激活需要将`flink-queryable-state-runtime_2.11-1.6.1.jar`从`opt/`拷贝到`lib/`下

### Making State Queryable
- 返回一个`QueryableStateStream`，类似一个Sink，incoming value是可查询的。比如`stream.keyBy(0).asQueryableState("query-name")`
- 调用`stateDescriptor.setQueryable(String queryableStateName)`将对应的keyed state置为可查询的
  ```
  ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
                "average", // the state name
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {})); // type information
  descriptor.setQueryable("query-name"); // queryable state name
  ```




# Operators

## Windows

**Keyed Windows**
```
stream
       .keyBy(...)               <-  keyed versus non-keyed windows
       .window(...)              <-  required: "assigner"
      [.trigger(...)]            <-  optional: "trigger" (else default trigger)
      [.evictor(...)]            <-  optional: "evictor" (else no evictor)
      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
       .reduce/aggregate/fold/apply()      <-  required: "function"
      [.getSideOutput(...)]      <-  optional: "output tag"
```

**Non-Keyed Windows**
```
stream
       .windowAll(...)           <-  required: "assigner"
      [.trigger(...)]            <-  optional: "trigger" (else default trigger)
      [.evictor(...)]            <-  optional: "evictor" (else no evictor)
      [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
      [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
       .reduce/aggregate/fold/apply()      <-  required: "function"
      [.getSideOutput(...)]      <-  optional: "output tag"
```

### Window Lifecycle
当属于这个window的第一个元素到达的时候窗口创建，当时间(event或者processing time)超过窗口的结束时间加上用户指定的`allowed lateness`则窗口被完全移除。
每个窗口会有一个Trigger和一个 window function(`ProcessWindowFunction`, `ReduceFunction`, `AggregateFunction` or `FoldFunction`)。window function定义对window中元素的操作，Trigger指定在什么条件下对窗口使用window function。Trigger还可以在窗口创建和删除之间purge一个窗口的内容(metadata还在，因此新数据还可以加入到窗口中)

### Keyed vs Non-Keyed Windows
keyed stream允许windowed计算被多个subtask并行执行，因为每个逻辑上的keyed stream可以独立处理。相同key的元素会被发送到相同的subtask
non-keyed streams只能被一个task处理，并行度为1

### Window Assigners
`WindowAssigner`用于定义元素如何分到对应的window(可能是一个或者多个)，Flink预定义了几个常用的assigner：`tumbling windows`, `sliding windows`, `session windows` and `global windows`，还可以自己定义。除了`global windows`另外三个都是基于时间(event time/processing time)来给窗口分配值。
基于时间的window都会有一个start time(包含)和一个end time(不包含)合起来描述窗口的大小，Flink使用`TimeWindow`来描述，`maxTimestamp()`返回窗口允许的最大的时间。

`Session Windows`没有固定的开始时间和结束时间，也不会重叠，当一段时间没有获取到元素则窗口关闭。这个时间可以是固定的Session gap，也可以是动态计算的Session gap。Session window operator为每条记录创建一个窗口，如果窗口之间的间隔小于gap则做merge。session window operator需要一个merging trigger和一个merging window function(FoldFunction无法使用)

`Global Windows`将相同key的元素放到一个window里，只有自定义了trigger才有意义，否则没有计算会发生，因为没有结束时间的概念。

### Window Functions
定义对窗口中的元素的计算逻辑。有四种`ReduceFunction`, `AggregateFunction`, `FoldFunction` or `ProcessWindowFunction`。前三种效率比较高，可以增量执行；最后一个获取所有元素的迭代器和窗口的元数据，需要缓冲所有的数据，可以通过和前三种配合达到增量聚合的效果，同时还可以使用窗口元数据。

`ProcessWindowFunction`还可以使用per-window state。有两种window:
- 当指定windowed operaton的时候定义的窗口，比如1小时的滑动窗口
- 指定key定义的窗口的一个实例，比如用户id为xyz从12点到13点的窗口。这是基于窗口的定义，根据key的数量会有很多的窗口。


per-windowstate指的是第二种。Context上定义了两个函数可以访问不同的state: 
- globalState()访问不属于当前窗口的对应key的状态
- windowState()访问属于当前窗口的对应key的状态


这个特性对于下面这种场景很有用：当数据延迟到达的时候需要late firing或者自定义trigger会进行speculative early firings，这时候会在per-window state上存储前一次fire的信息。当窗口清除的时候这个state也要删除。

### Triggers
定义一个窗口什么时候被window function处理。trigger接口提供了5个方法来处理不同的事件:
- The onElement() method is called for each element that is added to a window.
- The onEventTime() method is called when a registered event-time timer fires.
- The onProcessingTime() method is called when a registered processing-time timer fires.
- The onMerge() method is relevant for stateful triggers and merges the states of two triggers when their corresponding windows merge, e.g. when using session windows.
- Finally the clear() method performs any action needed upon removal of the corresponding window.

上面的方法有两点需要注意：
- 前面三个通过返回一个`TriggerResult`来决定如果对事件作出反应，反应的动作包括:
  - CONTINUE，不做任何事
  - FIRE，触发计算
  - PURGE，删除window中的数据
  - FIRE_AND_PURGE，触发计算，然后删除window的数据
- 这些方法都可以注册processing- 或者 event- timer

当trigger决定一个window可以被处理了，就会fire，即返回`FIRE`或者`FIRE_AND_PURGE`，这是让window operator发出当前window结果的信号。`ReduceFunction, AggregateFunction, or FoldFunction`只是简单将之前聚合的结果emit出去，`ProcessWindowFunction`会将所有的元素传给function。预定义的trigger都是返回`FIRE`，不会清除数据。

内置的trigger:
- EventTimeTrigger 基于watermark测量的event-time的进度fire，即当watermark超过window的结束时间的时候触发。是所有基于event-time的assigner的默认trigger
- ProcessingTimeTrigger 基于processing-time，是所有基于processing-time的assigner的默认trigger
- CountTrigger 当窗口中的元素超过一定数量的时候触发
- PurgingTrigger 以其他trigger作为参数，转成带purge功能的trigger

### Evictors
在一个trigger触发之后在window function执行之前或者之后从window移除数据。在window function之前evict的数据不会被window function处理。Flink有三个预定义的Evictor: 
- CountEvictor: 保留用户定义数量的元素，从window buffer的开头丢弃其他元素
- DeltaEvictor: 包含一个`DeltaFunction`和一个`threshold`，计算窗口中最近元素和buffer中其他元素的delta，超过阈值则移除
- TimeEvictor: 携带一个interval，找到所有元素中最大的时间戳 max_ts，移除所有时间戳小于(max_ts - interval)的数据

默认情况下，预定义的Evictor在window function之前执行逻辑。使用Evictor后会禁止所有的预聚合，计算之前所有的元素都要传递给Evictor。Flink对window中元素的顺序不做保证，因此上面的从头开始移除的数据是不确定的。

### Allowed Lateness
基于event-time处理的时候有可能元素会延迟到来，默认情况下当watermark超过窗口的结束时间的时候延迟到达的元素会被丢弃。Flink允许为window operator设置一个最大的`allowed lateness`，只有watermark超过window的结束时间加上`Allowed Lateness`元素才会被丢弃。再根据trigger的策略，延迟到达的元素还是有可能导致window fire，这是`EventTimeTrigger`的处理方式。

使用`side output`可以获取到因为延迟到达被丢弃数据。

### Working with window results
window操作的结果也是`DataStream`，不会有关于窗口的信息携带下去，如果要携带关于窗口的信息要使用`ProcessWindowFunction`自己定义。结果元素唯一携带的信息是元素的时间戳，即窗口的(end-time - 1)，这可以用于连续的窗口操作。

当watermark到达window operator的时候会触发两件事：
- 触发所有end-time - 1小于watermark的窗口进行计算
- watermark会发送到downstream operator


### state-size计算
- 每个window会保留一份数据，对于sliding window可能会有多份数据
- ReduceFunction, AggregateFunction, and FoldFunction会预先聚合，每个窗口只保存一个数据，但是如果单独使用`ProcessWindowFunction`则会保留所有数据
- 使用`Evictor`会阻止预聚合，需要保留所有数据


## Joining
### Window Join
window join 对相同key且相同窗口的两个stream进行join。这个窗口可以通过WindowAssigner定义。然后来自两个stream的数据传输到用户定义的`JoinFunction`或者`FlatJoinFunction`，满足join条件的数据会emit出去。通用的用法如下:
```
stream.join(otherStream)
    .where(<KeySelector>)
    .equalTo(<KeySelector>)
    .window(<WindowAssigner>)
    .apply(<JoinFunction>)
```
这里的join类似inter-join，即如果一个元素在另一个stream没有，就不会emit出去. emit出去的joined元素的timestamp等于window的end-time - 1

### Interval Join
对Stream A和B做join，其中 `b.timestamp ∈ [a.timestamp + lowerBound; a.timestamp + upperBound]`

![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/interval-join.svg)


## Process Function (Low-level Operations)

### ProcessFunction
low-level的stream操作，可以访问构建stream应用的基础部件：
- events (stream elements)
- state (fault-tolerant, consistent, only on keyed stream)
- timers (event time and processing time, only on keyed stream)

`ProcessFunction`可以看做是可以访问state和timer的`FlatMapFunction`，对input stream的每个event都会调用。通过`RuntimeContext`可以访问keyed state做fault tolerance。

timer可以应用对processing- 和 event- time的变化做出反应，每次对`processElement`方法的调用都会返回一个Context对象，用来访问元素的event-time和访问*TimeService*，后者可以用来为未来的processing- 或者 event- time注册callback，当时间到了，其`onTimer`方法被调用

### Low-level join
可以使用`CoProcessFunction`，对两个不同的输入分别调用`processElement1`和`processElement2`，实现low-level join一般使用下面的模式：
- 为一个输入创建一个state对象
- 当从一个输入收到一个元素的时候更新这个state对象
- 从另一个输入收到元素的时候产生joined结果

### KeyedProcessFunction
`ProcessFunction`的扩展，在`onTimer`方法中可以访问到timer的key

### Timers
processing- 和 event- time在内部都是由`TimerService`维护，排队等待执行。`TimerService`会为对应key和时间点的timer去重，onTimer方法只会调用一次。`onTimer`和`processElement`是同步的，不需要担心对状态的并发修改

Timer是fault tolerance的，会跟着应用的状态一起checkpoint，失败恢复或者从savepoint恢复的时候timer也会恢复。timer的checkpoint通常是异步的，除非结合使用` RocksDB backend / with incremental snapshots / with heap-based timers`。大量的timer会加重checkpoint的负担，可以对timer做合并。


## Asynchronous I/O for External Data Access
![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/async_io.svg)

比如访问外部的数据库，和数据库的交互时间可能占据了整个处理时间的大部分。正常情况下可以使用`MapFunction`来同步获取数据：发送一个请求，等待结果返回，但是这样大部分时间都花在等待结果返回。

和数据库的异步交互意味着一个方法实例可以并发处理很多请求，并发获取结果，因此等待事件可以分摊到多个请求上，可以获得比较高的吞吐量。使用的前提是外部系统有一个异步访问的客户端，如果没有也可以使用线程池来模拟异步请求，但是效率比较低。


# Connector 

## Kafka
**todo**

# Side Outputs
除了从`DataStream`中产生main stream，还可以产生任意数量的side output result stream，其类型可以和main stream不同，彼此之间也不需要相同。当需要把一个input stream分成多个stream的时候非常有用，否则就需要重复读取源数据，做filter。

使用过程：
- 首先定义一个`OutputTag`
  ```
  // this needs to be an anonymous inner class, so that we can analyze the type
  OutputTag<String> outputTag = new OutputTag<String>("side-output") {};
  ```
- emit数据
  ```
  DataStream<Integer> input = ...;
  
  final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};
  
  SingleOutputStreamOperator<Integer> mainDataStream = input
    .process(new ProcessFunction<Integer, Integer>() {
  
        @Override
        public void processElement(
            Integer value,
            Context ctx,
            Collector<Integer> out) throws Exception {
          // emit data to regular output
          out.collect(value);
  
          // emit data to side output
          ctx.output(outputTag, "sideout-" + String.valueOf(value));
        }
      });    
  ```
- 获取side output stream
  ```
  final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};
  SingleOutputStreamOperator<Integer> mainDataStream = ...;
  DataStream<String> sideOutputStream = mainDataStream.getSideOutput(outputTag);
  ```
  