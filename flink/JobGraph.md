# TODO
## Iteration
[Code](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/batch/index.html#iteration-operators)

[Introduction](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/batch/iterations.html)

## Visitor

## chain ?

## StreamTransformation
每个`DataStream`底层都有一个，是其起点，比如`DataStreamSource`从`SourceTransformation`开始。逻辑上的概念，和runtime的物理动作不一定一致。

# source/transformer/sink
## TypeInformation
Java的类型擦除导致类型信息丢失，flink中自己保存的类型信息，在创建source/transformer/sink的时候都会创建对应的类型信息。**创建的逻辑？**

### closer-clean
StreamExecutionEnvironment.clean **闭包清理的逻辑？**

## source
`DataStream`是相同类型的元素的stream，经过transform之后会产生其他的`DataStream`。除了预定义的还可以使用`StreamExecutionEnvironment.addSource`来指定第三方的source。有下面几种类型：

### SingleOutputStreamOperator
具有一种输出类型的流，大部分transform产生的都是这种流

### DataStreamSource
数据源

### IterativeStream
表示iteration流，调用`iterate`的时候产生

### KeyedStream
调用`keyBy`产生的包含key的流

### SplitStream
调用`split`产生的流，将流分成多个部分，通过select选择其中一个或者多个

### JoinedStreams

### CoGroupedStreams

### WindowedStream

## sink
`DataStreamSink`，调用`DataStream.addSink`产生

## transform
本质上是对一个或者多个Stream进行操作，返回一个新的stream。

# StreamGraph
在`StreamContextEnvironment.execute(String jobName) -> StreamExecutionEnvironment.getStreamGraph() -> StreamGraphGenerator.generate(this, transformations);`中生成`StreamGraph`.

表示一个streaming拓扑结构，包含构造`JobGraph`的所有信息。重要的数据结构：
```
private final StreamExecutionEnvironment environment;
private final ExecutionConfig executionConfig;
private final CheckpointConfig checkpointConfig;

private boolean chaining;

private Map<Integer, StreamNode> streamNodes;
private Set<Integer> sources;
private Set<Integer> sinks;
// SelectTransformation 产生
private Map<Integer, Tuple2<Integer, List<String>>> virtualSelectNodes;
// SideOutputTransformation 产生
private Map<Integer, Tuple2<Integer, OutputTag>> virtualSideOutputNodes;
// PartitionTransformation 产生
private Map<Integer, Tuple2<Integer, StreamPartitioner<?>>> virtualPartitionNodes;

protected Map<Integer, String> vertexIDtoBrokerID;
protected Map<Integer, Long> vertexIDtoLoopTimeout;
private StateBackend stateBackend;
private Set<Tuple2<StreamNode, StreamNode>> iterationSourceSinkPairs;
```

## StreamNode
表示operator和它的所有属性，比如并行度，还有输入边`inEdges`和输出边`outEdges`

## StreamEdge
连接两个`StreamNode`的边，由于`chaining`或者其他优化，执行的时候不一定对应一条实际的连接。

## StreamGraphGenerator
`StreamGraphGenerator`从`StreamTransformation`构建出`StreamGraph`。`StreamExecutionEnvironment`有个`transformations`字段，在main函数中类似map/filter这个的transform本质上只是往这个字段里加一个`StreamTransformation`(包含了上一步输入)；而keyBy/split之类的transform会构造一个新的Stream，包含了input stream。在执行execute的时候这个字段已经包含了所有的转换信息。因此构造`StreamGraph`的逻辑如下：
从sinks开始遍历`transformations`，对每个`transformation`递归遍历其inputs，然后在`StreamGraph`中创建一个node，然后从inputs到新创建的node添加edges。一个典型的转换代码如下:
```
private <IN, OUT> Collection<Integer> transformOneInputTransform(OneInputTransformation<IN, OUT> transform) {
     
     //递归遍历inputs
	Collection<Integer> inputIds = transform(transform.getInput());

	// the recursive call might have already transformed this
	if (alreadyTransformed.containsKey(transform)) {
		return alreadyTransformed.get(transform);
	}

	String slotSharingGroup = determineSlotSharingGroup(transform.getSlotSharingGroup(), inputIds);

     // 这里会添加实际的节点
	streamGraph.addOperator(transform.getId(),
			slotSharingGroup,
			transform.getCoLocationGroupKey(),
			transform.getOperator(),
			transform.getInputType(),
			transform.getOutputType(),
			transform.getName());

	if (transform.getStateKeySelector() != null) {
		TypeSerializer<?> keySerializer = transform.getStateKeyType().createSerializer(env.getConfig());
		streamGraph.setOneInputStateKey(transform.getId(), transform.getStateKeySelector(), keySerializer);
	}

	streamGraph.setParallelism(transform.getId(), transform.getParallelism());
	streamGraph.setMaxParallelism(transform.getId(), transform.getMaxParallelism());

     // 从inputs到当前input创建edges
	for (Integer inputId: inputIds) {
		streamGraph.addEdge(inputId, transform.getId(), 0);
	}

	return Collections.singleton(transform.getId());
}
```

对于`Partitioning, split/select and union`，在`StreamGraph`中没有产生实际的node，用`virtual node`来承载相应的属性：
```
private <T> Collection<Integer> transformPartition(PartitionTransformation<T> partition) {
	StreamTransformation<T> input = partition.getInput();
	List<Integer> resultIds = new ArrayList<>();
	Collection<Integer> transformedIds = transform(input);
	for (Integer transformedId: transformedIds) {
		int virtualId = StreamTransformation.getNewNodeId();
         // 这里添加虚拟节点
		streamGraph.addVirtualPartitionNode(transformedId, virtualId, partition.getPartitioner());
		resultIds.add(virtualId);
	}
	return resultIds;
}
```

# JobGraph
在`StreamContextEnvironment.execute(String jobName) -> ctx.getClient().run -> ClusterClient.getJobGraph -> StreamingPlan.getJobGraph() -> StreamingJobGraphGenerator.createJobGraph(this, jobID);`中将一个`StreamGraph`转成`JobGraph`

`JobGraph`代表一个JobManager接受的实际的Flink dataflow程序，所有高级的api最终都是转成`JobGraph`。`JobGraph`是一个vertices和中间结果组成的图，彼此互相连接组成一个DAG。最重要的数据结构是`Map<JobVertexID, JobVertex> taskVertices`，另外还有其他配置比如`ExecutionConfig`/`JobCheckpointingSettings`/`SavepointRestoreSettings`这些常规配置，以及执行期间需要的资源`userJars`/`userArtifacts`/`classpaths`等。

## JobVertex
`JobGraph`的顶点，包括包含的operator `operatorIDs`，输入`inputs`，中间结果`results`。可以理解为`operatorIDs`对`inputs`进行操作，产生`results`。主要属性：
```
	/** The ID of the vertex. */
	private final JobVertexID id;

	/** The alternative IDs of the vertex. */
	private final ArrayList<JobVertexID> idAlternatives = new ArrayList<>();

	/** The IDs of all operators contained in this vertex. */
	private final ArrayList<OperatorID> operatorIDs = new ArrayList<>();

	/** The alternative IDs of all operators contained in this vertex. */
	private final ArrayList<OperatorID> operatorIdsAlternatives = new ArrayList<>();

	/** List of produced data sets, one per writer */
	private final ArrayList<IntermediateDataSet> results = new ArrayList<IntermediateDataSet>();

	/** List of edges with incoming data. One per Reader. */
	private final ArrayList<JobEdge> inputs = new ArrayList<JobEdge>();

	/** Number of subtasks to split this task into at runtime.*/
	private int parallelism = ExecutionConfig.PARALLELISM_DEFAULT;

	/** Maximum number of subtasks to split this task into a runtime. */
	private int maxParallelism = -1;

	/** The minimum resource of the vertex */
	private ResourceSpec minResources = ResourceSpec.DEFAULT;

	/** The preferred resource of the vertex */
	private ResourceSpec preferredResources = ResourceSpec.DEFAULT;

	/** Custom configuration passed to the assigned task at runtime. */
	private Configuration configuration;

	/** The class of the invokable. */
	private String invokableClassName;

	/** Indicates of this job vertex is stoppable or not. */
	private boolean isStoppable = false;

	/** Optionally, a source of input splits */
	private InputSplitSource<?> inputSplitSource;

	/** The name of the vertex. This will be shown in runtime logs and will be in the runtime environment */
	private String name;

	/** Optionally, a sharing group that allows subtasks from different job vertices to run concurrently in one slot */
	private SlotSharingGroup slotSharingGroup;

	/** The group inside which the vertex subtasks share slots */
	private CoLocationGroup coLocationGroup;

	/** Optional, the name of the operator, such as 'Flat Map' or 'Join', to be included in the JSON plan */
	private String operatorName;

	/** Optional, the description of the operator, like 'Hash Join', or 'Sorted Group Reduce',
	 * to be included in the JSON plan */
	private String operatorDescription;

	/** Optional, pretty name of the operator, to be displayed in the JSON plan */
	private String operatorPrettyName;

	/** Optional, the JSON for the optimizer properties of the operator result,
	 * to be included in the JSON plan */
	private String resultOptimizerProperties;
```

## JobEdge
连接上游的中间结果和下游的`JobVertex`的边(communication channel):
```
	/** The vertex connected to this edge. */
	private final JobVertex target;

	/** The distribution pattern that should be used for this job edge. */
	private final DistributionPattern distributionPattern;
	
	/** The data set at the source of the edge, may be null if the edge is not yet connected*/
	private IntermediateDataSet source;
	
	/** The id of the source intermediate data set */
	private IntermediateDataSetID sourceId;
	
	/** Optional name for the data shipping strategy (forward, partition hash, rebalance, ...),
	 * to be displayed in the JSON plan */
	private String shipStrategyName;

	/** Optional name for the pre-processing operation (sort, combining sort, ...),
	 * to be displayed in the JSON plan */
	private String preProcessingOperationName;

	/** Optional description of the caching inside an operator, to be displayed in the JSON plan */
	private String operatorLevelCachingDescription;
	
```

## IntermediateDataSet
一个operator产生的中间结果集，可以被其他operator消费、materialized或者被丢弃。包含产生这个数据集的`JobVertex`和消费这个数据集的`List<JobEdge>`以及结果类型:
```
	private final JobVertex producer;			// the operation that produced this data set
	private final List<JobEdge> consumers = new ArrayList<JobEdge>();
	// The type of partition to use at runtime
	private final ResultPartitionType resultType;
```

## StreamingJobGraphGenerator.createJobGraph
最重要的是生成`JobGraph`的`Map<JobVertexID, JobVertex> taskVertices`结构。会产生多少个`JobVertex`和**chaining**的策略有关。比如下面的代码:
```
DataStream<String> input = env
	.fromElements("a", "b", "c", "d", "e", "f")
	.filter(v -> v.compareTo("a") > 0)
	.map(v -> v + "z")//.startNewChain()
	.filter(v -> v.equals("ab"))
	.map(v -> v + "y");
input.addSink(new SinkFunction<String>() {
	@Override
	public void invoke(String value) throws Exception {}
});
```
- 如果没有禁用**chaining**的功能，flink会尽量使用，上面最终只会生成一个`JobVertex`:`fromElements->filter->map->filter->map->sink`
- 如果把`startNewChain`的注释去掉，则从这个map开始分为前后两个chain，因此会有两个`JobVertex`，第一个`fromElements->filter`，第二个`map->filter->map->sink`。

`createJobGraph`中处理**chaining**功能的代码在`createChain`中，对每个`source`调用这个方法去创建`JobVertex`。这是一个递归方法，首先将是否能**chaining**将`outEdges`分组：
- 如果可以**chaining**则递归调用，其中`startNodeId`不变，`currentNodeId`取outEdge的target，且结果放入`transitiveOutEdges`
- 否则先将outEdge`放入`transitiveOutEdges`，在递归调用，且`startNodeId`和`startNodeId`都是这个outEdge的target

`transitiveOutEdges`用于过渡使用的`outEdge`，可以理解为开始一个新的chain那个operator对应的上一条`StreamEdge`，比如上面的例子中就是第一个map到第一个filter的那条`StreamEdge`。

经过上面这两步创建出来的是彼此没有连接的`JobVertex`，需要通过`JobEdge`连接起来，这是`connect`方法的作用，当当`startNodeId`和`currentNodeId`相等的时候会遍历`transitiveOutEdges`，从`startNodeId`对应的`JobVertex`创建到`transitiveOutEdges`对应的`JobVertex`对应的`JobEdge`。

比如对上面的operator编号从1-6，按方法中`startNodeId`和`currentNodeId`的调用顺序来看(区别在编号为3的那个operator上):
- (1,1)->(1,2)->(1,3)->(1,4)->(1,5)->(1,6)
- (1,1)->(1,2)->(3,3)->(3,4)->(3,5)->(3,6)
