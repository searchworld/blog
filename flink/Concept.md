# Concept
## Programming Model
### Level of abstraction
![Programming Model](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/levels_of_abstraction.svg)
 表达能力从下往上增强，但是灵活性不断降低。

- Stateful Stream Processing
- DataStream/DataSet
- Table API，声明式定义逻辑操作，可以和DataStream/DataSet混用
- SQL

### Programs and Dataflows
flink程序的基础block是`stream`和`transform`，执行的时候会映射到streaming dataflows，由stream和transform operator组成。每个dataflow由source开始，到sink结束。dataflow组成任意的directed acyclic graphs(DAGs)组成。通过`iteration`可以构造出特殊类型的循环图，不过主要还是DAG

### Parallel Dataflows

![Parallel Dataflows](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/parallel_dataflow.svg)

每个stream可以有多个的`stream partition`，每个operator有一个或者多个（这个数量就是operator的`parallelism`）的`operaton subtasks`，subtask在不同的线程中执行，互不干扰。

stream的并行度就是producing operator的并行度，同一个程序的不同operator可以有不同层级的并行度。

两个operator直接的数据传输可以有两种模式：
- one-to-one模式，保留元素的分区和顺序
- Redistributing模式，改变元素的分区，比如keyBy操作

### Windows
可以是时间驱动(30s内的数据和)或者数据驱动(每100个元素的和)

### Time
Event time/Ingestion time/Processing time

### Stateful Operations

![Stateful Operations](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/state_partitioning.svg)

类似window operators会记录跨多个事件的消息，称为有状态的。状态认为是维护在嵌入的kv数据库里。这些状态在stream被stateful operator读取的时候会跟stream一起分区和分发。stateful operator只能用在keyed stream上

### Checkpoints for Fault Tolerance
flink结合stream replay和checkpointing来实现fault tolerance。Checkpoint是input stream的一个特殊点和operator对应的状态。通过恢复operator的状态，replay checkpoint开始的事件，streaming dataflow可以从一个checkpoint恢复并维持一致性。

### Batch on Streaming
stream的一种特殊case，stream是有限的。有几个主要的区别：
- Fault tolerance。由于数据是有限的，直接使用replay，不需要checkpoint
- stateful operation. 直接使用简化的in-memory/out-of-core数据结构，而不是kv数据库
- DataSet API引入了特殊的`synchronized iteration`，对于bounded stream才可以使用。

## Distributed runtime
### Tasks and Operator Chains

![Chains](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/tasks_chains.svg)

分布式执行的时候flink将subtasks 链接成tasks，每个task在一个线程中执行。chaining是个很有用的优化：减少线程之间的交互的开支，提供吞吐率，降低延迟。是可配置的。

### Job Managers, Task Managers, Clients

![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/processes.svg)

flink运行时包含两种进程：
- **JobManager**(也称为master)，协调分布式执行，比如调度任务，协调checkpoint和失败恢复。至少有一个，如果做HA可以有多个。
- **TaskManagers**(也称为workers)，负责执行task，缓存和交换stream。至少要有一个。

**Client**用于向JobManager发送dataflow，然后就可以断开，也可以继续连着获取状态。

### Task Slots and Resources

![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/tasks_slots.svg)

task slot用来控制一个worker接收多少个task，每个task slot代表TaskManager资源的一部分，因此不同TaskManager的task slot不会彼此竞争资源。当前slot只区分内存，CPU还是所有都共享。



同一个JVM中的task共享TCP连接和心跳信息、数据集和数据结构，减少每个task的开支。
默认情况下即使不同task的subtask可以共享slot，只要是来自同一个job。因此一个slot可能包含job的整个pipeline，slot共享有几个好处：
- Flink集群只需要最高的并行度作为task slot的数量，不需要计算一个程序需要多少tasks
- 更好的利用资源。如果不使用slot sharing，非资源密集型的subtask(比如map)会像资源密集型的subtask(比如window) block一样多的资源。
  ![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/slot_sharing.svg)

作为经验值，默认的task的数量是CPU核的数量，对于超线程，每个slot有两个或者更多的线程

### State Backends
![](https://ci.apache.org/projects/flink/flink-docs-release-1.6/fig/checkpoints.svg)

目前有两种，一种是保存在内存中的hashmap，另一种是RocksDB. 除了定义维护状态的数据结构，state backend还实现了snapshot和保存snapshot作为checkpoint一部分的逻辑。

### Savepoints
人工触发的checkpoint，获取程序的一个快照，并写到state backend。checkpoint是定时产生，且只有最后一份才是有效的，之前的都会失效。savepoint不会自动失效