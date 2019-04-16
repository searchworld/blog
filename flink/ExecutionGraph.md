# component
`ExecutionGraph`是协调data flow分布式执行的中心数据结构，保存了每个并行task、每个intermediate stream以及他们之间的通信。主要包含以下几种组件：
- `ExecutionJobVertex`表示执行过程中`JobGraph`的一个顶点，保存了所有并行subtask的聚合状态。
- `ExecutionVertex`表示一个并行的task，一个`ExecutionJobVertex`中`ExecutionVertex`的数量等于并行度
- `Execution`是一次执行`ExecutionVertex`的尝试，当失败或者数据不存在需要重新计算的时候可能会有多个

## failover
`ExecutionGraph`有两种failover模式:
- `global failover`中断所有vertex的`Execution`，从上次完成的checkpoint重启整个dataflow，可以任务是`local failover`不成功或者状态异常(比如因为bug)的fallback策略
- `local failover`在一个单独的`Execution`失败的时候触发，使用`FailoverStrategy`来协调

在local- 和 global-之间，后者的优先级更高，因为后者是保证一致性的核心机制。`ExecutionGraph`保存一个`global modification version`，每次`global-failover`的时候递增。如果这两者并发执行，failover strategy需要保证local-先停下来。

# Data structure
## ExecutionGraph
```
/** All job vertices that are part of this graph. */
private final ConcurrentHashMap<JobVertexID, ExecutionJobVertex> tasks;

/** All intermediate results that are part of this graph. */
private final ConcurrentHashMap<IntermediateDataSetID, IntermediateResult> intermediateResults;

/** The currently executed tasks, for callbacks. */
private final ConcurrentHashMap<ExecutionAttemptID, Execution> currentExecutions;

/** The implementation that decides how to recover the failures of tasks. */
private final FailoverStrategy failoverStrategy;

/** The slot provider to use for allocating slots for tasks as they are needed. */
private final SlotProvider slotProvider;

/** On each global recovery, this version is incremented. The version breaks conflicts
 * between concurrent restart attempts by local failover strategies. */
private volatile long globalModVersion;
```

## ExecutionJobVertex
```
// 包含的operator
private final List<OperatorID> operatorIDs;

// parallelism个并发的subtask
private final ExecutionVertex[] taskVertices;

// 产生的中间数据
private final IntermediateResult[] producedDataSets;

// 输入
private final List<IntermediateResult> inputs;

private final int parallelism;
```

## ExecutionVertex
```
private final Map<IntermediateResultPartitionID, IntermediateResultPartition> resultPartitions;

// 由于并行度的存在，这里边有可能是(N * M)条
private final ExecutionEdge[][] inputEdges;

/** The current or latest execution attempt of this vertex's task. */
private volatile Execution currentExecution;
```

## Execution
```
/** The executor which is used to execute futures. */
private final Executor executor;

/** The unique ID marking the specific execution instant of the task. */
private final ExecutionAttemptID attemptId;
```

# rumtime
## build
```
JobMaster(...) 
  this.jobManagerJobMetricGroup = jobMetricGroupFactory.create(jobGraph);
  this.executionGraph = createAndRestoreExecutionGraph(jobManagerJobMetricGroup);
    createExecutionGraph(currentJobManagerJobMetricGroup);
      ExecutionGraphBuilder.buildGraph(...)
        new ExecutionGraph(...) // 创建一个新的ExecutionGraph
        executionGraph.attachJobGraph(sortedTopology);
          ExecutionJobVertex ejv = new ExecutionJobVertex(...) // 针对每个JobVertex创建一个ExecutionJobVertex
            new ExecutionVertex(...) // 按并行度创建ExecutionVertex
              new Execution(...) // 对每个ExecutionVertex创建Execution
          ejv.connectToPredecessors(this.intermediateResults); // 在ExecutionJobVertex之间创建ExecutionEdge
        // 处理checkpoint
        executionGraph.enableCheckpointing(...)
    // 处理savepoint
    tryRestoreExecutionGraphFromSavepoint(newExecutionGraph, jobGraph.getSavepointRestoreSettings());
```

## schedule
`TaskManager`是旧的方式，yarn方式对应的子类是`YarnTaskManager`，在`YarnApplicationMasterRunner.getTaskManagerClass`中使用到，在容器中启动的时候跟着启动，已经没用了。

`TaskExecutor`是新版实现:
```
YarnResourceManager.onContainersAllocated
  createTaskExecutorLaunchContext
    Utils.createTaskExecutorContext(...) // 这里使用YarnTaskExecutorRunner
  nodeManagerClient.startContainer(container, taskExecutorLaunchContext); // 在每个container启动的时候会启动YarnTaskExecutorRunner

YarnTaskExecutorRunner.main
  run -> TaskManagerRunner.runTaskManager
    new TaskManagerRunner(configuration, resourceId);
      taskManager = startTaskManager(...) // 这里创建TaskExecutor，这个命名有点奇葩
    taskManagerRunner.start();
      taskManager.start();
```

最终提交任务直接使用`TaskExecutor.submitTask`: 
```
ExecutionGraph.scheduleEager(SlotProvider slotProvider, final Time timeout)
  ejv.allocateResourcesForAll(...) // 对每个ExecutionJobVertex申请资源
    exec.allocateAndAssignSlotForExecution(...) // 为每个Execution申请slot
      slotProvider.allocateSlot(...) // 这里的slotProvider是SlotPool.ProviderAndOwner
      tryAssignResource(logicalSlot)
    execution.deploy(); // 上面分配完资源，这里把execution部署到资源上
      TaskDeploymentDescriptor deployment = vertex.createDeploymentDescriptor(...)
      taskManagerGateway.submitTask(deployment, rpcTimeout); // RpcTaskManagerGateway
        taskExecutorGateway.submitTask(tdd, jobMasterId, timeout); // taskExecutorGateway是TaskExecutor的代理，在JobMaster.registerTaskManager生成

TaskExecutor.submitTask
  task = new Task(...)
  task.startTaskThread()
    Task.run()
      invokable = loadAndInstantiateInvokable(userCodeClassLoader, nameOfInvokableClass, env);
      invokable.invoke(); -> StreamTask.invoke()
        operatorChain = new OperatorChain<>(this, streamRecordWriters); // 拿到这个Task所有的operator
        init() // 执行task自己的初始化逻辑
        initializeState(); // 初始化状态
        openAllOperators(); // 打开operator
        run(); // 执行task自己的运行逻辑
        closeAllOperators(); // 关闭所有的operator
        tryDisposeAllOperators();        
```

### OperatorChain
包含了所有被作为一个chain在`StreamTask`中执行的operator，需要从一个chain的整体去处理输入，因此第一个operator会比较特殊，需要负责上游的输入
```
private final StreamOperator<?>[] allOperators;
// 负责输出
private final RecordWriterOutput<?>[] streamOutputs;
// 负责输入
private final WatermarkGaugeExposingOutput<StreamRecord<OUT>> chainEntryPoint;
// 第一个operator
private final OP headOperator;
```

# StreamTask
实现`AbstractInvokable`，是所有streaming task的基类。task是`TaskManager`部署和执行的本地处理单元，会运行一个或者多个`StreamOperator`，组成这个operator chain: 包含一个head operator和多个chained operators，head operator的类型指定了这个`StreamTask`的类型: one-input / two-input / source / iteration heads / iteration tails

## SourceStreamTask
用于执行`StreamSource`，run方法最终调用`StreamSource.run`

# SchedulingStrategy
根据`CheckpointingOptions.LOCAL_RECOVERY`参数分为两种：
- 如果参数为false(默认情况)，则使用`LocationPreferenceSchedulingStrategy`，即根据位置偏好来分配slot。**偏好的匹配逻辑?**
- 否则使用`PreviousAllocationSchedulingStrategy`，是`LocationPreferenceSchedulingStrategy`的子类，尽量使用之前分配过的slot

# SlotManager & SlotProvider
