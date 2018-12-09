# Chapter 6. Streams and Tables
第一部分讲的是`cardinality`(基数，即数量，对应bounded和unbounded)，这部分开始讲`constitution`(组成方式，即stream和table)。从`cardinality`到`constitution`的切换有点类似经典物理学和量子力学的比较。

## Stream-and-Table Basics Or: a Special Theory of Stream and Table Relativity
S&T的基本思想来自数据库，熟悉SQL的都会熟悉table及其属性：table包含行列数据，每行都通过一个唯一key唯一标识。数据库的底层数据结构是一个`append-only log`，随着transaction被用在数据库上，这些transaction先记录到一个log上，然后再作用到数据库上将更新materialize。在S&T上面这个log就是stream。从这个角度我们可以知道如何从stream创建table: **应用stream中的更新的transaction log的结果就是table**。从table创建stream: **stream是table的changelog。**table-to-stream的典型例子是`materialized views`，即指定table上的一个query，数据库将其视为另一个first-class table，本质上是query的一个cache，系统保证其内容随着table内容的变化跟着变化，通过跟踪源表的changelog，每次将变化用到`materialized view`上。结合这两点可以得到Stream和Table的相对论:
- Streams → tables，对update的stream进行聚合得到table
- Tables → streams，对一个table的change的观察得到stream

### Toward a General Theory of Stream and Table Relativity
上面的Stream和Table的相对论还有几个问题要解决：
- 如何将这个相对论应用到batch system中
- stream对于有限和无限数据集有什么关系？
- 如何将what/where/when/how映射到S&T中

关于T&S的解释：
- Tables are data at rest。并不是说Table是静止的，大部分的table都是随着时间变化的，但是在某个时间点，table的一个snapshot提供了所包含的所有数据的一个整体图像，因此table提供了一个供数据进行accumulation的地方，并可以按时间进行观察
- Streams are data in motion。stream捕获数据随着时间的变化情况。可以认为stream是table的微分，table是stream的积分。stream捕获了table中数据的移动。

## Batch Processing Versus Streams and Tables
### A Streams and Tables Analysis of MapReduce
这里讨论如何将S&T理论应用到MapReduce。MapReduce由Map和Reduce组成，实际上又可以分成6个部分：
- MapRead，读入输入数据，预处理成K/V结构的作为Map的输入
- Map，从上一步获取K/V，输出0或者多个K/V值
- MapWrite，将上一阶段输出的相同Key的值做聚合，一key/value-list的格式写入持久化存储，这一步实际上市group-by-key-and-checkpoint的过程
- ReduceRead，消费上一步shuffle保存下来的数据，转成key/value-list的格式给下一个阶段
- Reduce，重复消费一个key和其对应的value-list，输出0或者多个值，可以仍然和key保持关系(可选)
- ReduceWrite，将上一步的输出写入output datastore

MapWrite和ReduceRead通常称为Shuffle，现在更多的是对应到Source和Sink。

- Map as streams/tables。

    MapRead是迭代input table中的数据，并以流的方式将数据变成运动的，供Map消费。Map过程只是将一个Stream转成另一个Stream。MapWrite的是否写入持久化存储并不重要，关键的是groupBy操作，符合上面Stream to table的的定义：将一个update的流做Aggregation，因此MapWrite将Stream转成Table。

- Reduce as streams/tables

    ReduceRead类似MapRead，Reduce类似Map。当ReduceWrite的输出和key相关的时候也和MapWrite一样，但是如果输出和key无关的话，ReduceWrite会将每个记录当做有一个唯一key来看待。这里的key很关键，类似SQL中的主键，是必须的。

### Reconciling with Batch Processing
上面的过程回答了第一个问题：如何将Batch系统应用到stream/table理论，即table-stream-stream-table-stream-stream-table。

第二个问题stream和bound/unbounded 数据的关系，从上面的过程也可以看到，stream只是运动中的数据，和bound/unbounded没有直接关系

从这stream/table的角度看，batch和stream系统并没有本质区别

## What, Where, When, and How in a Streams and Tables World
### What: Transformations
从S&T的角度，实际上有两种类型的transform:
- Nongrouping，类似Map和Reduce，接收一个Stream，产生另一个stream，不会阻止stream中元素的motion。
- Grouping，类似MapWrite和ReduceWrite，接收一个stream，按某种形式做group，将stream转为table，会将元素从motion变成rest

将stream processor作为一个database最重要的点：pipeline中任何一个有group操作的地方都是在创建一个table，包含那个阶段对应部分的输出。这是kafka和flink正在努力的方向。如果这个output value刚好是pipeline正在计算的值，且可以从table直接读取，那就没必要在其他地方 rematerialize。这种方式提供了对变化的结果的快速访问，同时可以节省一次sink阶段计算资源和减少多余的数据存储。唯一需要关注的是要保证只有pipeline才能去修改这个table，否则就没法保证数据一致性。(这里应该是对应flink的queryable state)

### Where: Windowing
- window assignment，当把一个record放入一个window的时候，实际上window的定义和record的key在group的时候组成了一个隐式的composite key，即key+window组合
- window merging，在assignment的时候可以把隐式的key看做是用户定义的key和window的铺平组合，但是在需要merge的时候，group操作需要考虑所有可能需要merge的window，这个时候会将composite key看做是层次结构，root层是用户定义的key，下一层才是window。做group的时候先对用户定义的key做group，然后再处理key下面的window

从S&T的角度还要考虑如何随着时间修改changelog。对于nonmerging window，每个新的元素只会对table产生一个mutation。对于merging window，group的时候可能会导致一个或者多个已经存在的window合并到新的window，因此merge操作需要检查当前key所有已经存在的window，计算哪些window可以合并，然后产生还没有合并的window的delete操作和新window的insert操作。因此支持window merging的系统通常定义用户指定的key作为`the unit of atomicity/parallelization`，而不是key+window，否则就无法(或者代价非常大)提供正确性保证需要的强一致性。

到这里可以看到，window操作只需要对group做出微小的改动，即stream-to-table的微小变动。

### When: Triggers
在S&T语境下，trigger是一个特殊的过程，应用到table上，当相关的事件发生的时候允许table中的数据被物化。这听起来很像数据库的触发器，事实上trigger的名字就是从触发器产生的。当指定了trigger，实际上是写了一段代码，每一行数据都会被evaluated，当这个trigger被fire的时候，会将原来已经在table中的相关数据又变成运动的，产生一个新的stream。

对于batch系统，trigger只有一种：当输入结束的时候触发。当把其他trigger用在batch system上的时候有几点需要讨论: 
- Trigger guarantees。现在大部分的batch系统都是按`lock-step read-process-group-write-repeat`的顺序设计，因此基本没法提供细粒度的trigger能力，因为唯一能操作的只有shuffle环节。trigger提供的保证是trigger发生之后，而不是trigger发生的时候。比如`AfterCount(N)`意味着超过N之后会触发，但是要N之后的多久是不确定的。这对模型本身是非常需要的，因为trigger本身的异步性和不确定性。
- The blending of batch and streaming。trigger在batch和stream系统上的语义区别在于后者可以增量触发，但是本质上这也不是一个语义区别，更多的是latency/throughput trade-off：batch系统高吞吐量高延迟，stream低吞吐量低延迟。batch系统的高吞吐量来自于更大bundle size和优化过的shuffle过程。因此可以提供一个系统结合这两者的优势。Beam API在上层做了统一，但是execution-engine level也同样可以得到统一。在这样的系统中就不再有batch和stream的概念，全部都是数据处理。

### How: Accumulation
S&T理论对accumulation没有什么改动。

## A General Theory of Stream and Table Relativity
- 数据处理pipeline由table/stream/operation组成
- Tables are data at rest，为数据accumulation和随时间的变化被观测提供容器
- Streams are data in motion，提供了table随时间进化的离散化视图
- Operation操作一个table或者stream，产生一个新的table或者stream，如下分类：
    - stream -> stream: nongrouping operation
    - stream -> table: grouping operation
        - windowing将时间维度包含到group里面
        - merging window随着时间动态结合，允许随着数据被观察到对window进行reshape，并指定key作为atomicity/parallelization的单位，window作为那个key做group的时候的child component
    - table -> stream: Ungrouping (triggering) operations，将数据转成in motion，产生一个stream，捕获table随时间变化的视图
        - `Watermark`提供相对于event-time的输入完整性概念
        - `accumulation mode`决定了stream的类型，可以是包含delta，也可以只value，甚至是之前值的retraction
    - table -> table: (none)


# Chapter 7. The Practicalities of Persistent State
persistent state实际上就是第6章讨论的table，加上一个要求：必须是存储在不容易丢失的介质中。

## Motivation
- The Inevitability of Failure，对于bounded DataSet更倾向于重新开始处理
- Correctness and Efficiency
    - 保证ephemeral inputs正确性的基础，当机器发生失败的时候可以从这个persistent state恢复，不管上游数据是否已经发生变化
    - 最小化重复性工作和需要持久化的数据

## Implicit State
persistent state的最佳实践实际上是always persist everything 和 never persist anything中做权衡。

### Raw Grouping
row grouping类似list appending：每次有新元素到达这个group，就追加到这个group的list末尾。输入是一个<K,V>，返回<K,Iterable<V>>。存在的问题：
- 需要存储非常多的数据
- 如果有多个trigger触发可能会导致数据重复
- checkpoint数据多，恢复慢

### Incremental Combining
incremental combining的基础是许多Aggregation有下面的属性：
- 有一个中间形式，表示compact N个输入的partial progress，比这些输入的full list更加紧凑，比如求均值的时候只要保持sum和count
- combine操作要满足交换律和结合律，意味着我们可以按照任意顺序来做聚合。可以通过两种方式来优化：
    - Incrementalization，由于顺序无关，每次有新元素过来就可以直接combine，而不用维护一个有序集合，这样减少了数据量，同时也将负载分配的比较平均
    - Parallelization，可以现在某些机器上做subgroup，最后再做聚合，因此可以把计算分配到多台机器上。可并行说明可以兼容merging window，在做merge的时候只需要O(1)的复杂度

## Generalized State
raw和incremental grouping不够灵活，如果要支持更通用的persistent state，需要有三个维度的灵活性：
- 数据结构。Raw grouping本质上是一个appended list，incremental combining是single value。但是仍然需要类似map tree graph set等其他灵活的数据结构，每种数据结构都有自己的访问模式和cost。
- 读写颗粒度。以最优效率读写数据的能力，即在给定时间内只读取需要的数据。这和上一条数据结构的访问模式紧密结合，比如Set可以结合bloom过滤器；另外还可以通过异步batch的方式提供效率
- 处理过程的调度，即可以和event-time或者processing-time结合。trigger在这里提供了一部分灵活性，completeness trigger将processing和watermark经过window的结尾结合起来，repeated update trigger将processing和processing-time的周期进度结合起来，但是对于有些类型灵活性还是不够。在Beam中通过`timer`来提供处理调度的灵活性，这是一个特殊的state，绑定event-time或者processing-time的某一个时间点，当到达这个时间点的时候触发一个方法，去处理具体的逻辑

### Case study
Conversion Attribution

# Chapter 8. Streaming SQL
## What Is Streaming SQL?
这章很多讨论还是假想的或者系统实现的理想情况，很多功能在很多系统上都还没有实现。而且这里的讨论很多来源于`Calcite`/`Flink`/`Beam`社区的讨论，本章主要提炼出这次讨论的主要观点。

### Relational Algebra
关系代数是SQL的理论基础，是用来描述有名称有类型的元组的数据之间关系的数学方式，其核心是relation本身，即元组的一个集合。在典型的数据库中，relation是一个数据库表或者查询的结果。

关系代数更重要的属性是闭包属性：对合法的relation使用关系代数中的任何操作会产生另一个relation，换句话说relation是关系代数的硬通货，所有操作都是消费relation，产生relation。历史上对stream SQL的支持最后失败都是因为不满足这个闭包属性，将stream和传统的relation区别对待，提供新的操作对这两个进行转化，并限制一些操作对stream使用。同时这些系统对一些关键属性不支持，比如乱序数据。因此其使用率不高。如果要让stream SQL推广起来，必须让streaming称为一等公民


### Time-Varying Relations
将streaming融入SQL的关键是扩展关系代数的核心-relation，用来表示随着时间变化的数据集，而不是在某个时间点的数据集；即我们需要`time-varying relations`而不是`point-in-time relations`。从关系代数的角度，`time-varying relations`是经典的relation随时间进化。比如用户事件数据，随着时间过去，用户产生新的事件，数据集持续增加和进化。如果在某个时间点去观察，那就是classic relation；但是如果随着时间观察数据集，就是`time-varying relations`。从另一个角度，如果classic relation是二维坐标，那`time-varying relations`就是多了时间轴的三位坐标。随着relation的变化，在时间轴上都会有snapshot。

![TVR](https://github.com/searchworld/blog/blob/master/image/TVR.png)

这里将时间维度铺平，变成4个不同的小窗口，每个窗口对应一个time range内的数据，互相之间时间是连续的。从图上可以看出，`time-varying relations`只是一系列的classic relations，每个relation独立存在于相邻的一个时间范围，在这个时间范围内这个relation是不变的。这意味着对TVR使用relational operator等价于按顺序对每个classic relation独立使用relational operator。更进一步，独立对关联时间区间的classic relations使用relational operator会产生相同时间区间的relation序列，即结果是一个响应的TVR。这里给出了两个很重要的属性：
- classic relational algebra的所有operator对TVR仍然有效，且仍然能得到预期结果
- 关系代数的`closure property`应用到TVR的时候仍然是完好的

简洁的表述就是：classic relation的所有规则对TVR仍然是一样的。因此对SQL并不需要什么大的修改，只需要将之前应用到单个relation的方式变成应用到一个序列的TVR

上面定义的TVR更多是理论上的，数据很容易就膨胀变大，对于频繁变化的大数据集也不是很方便。这个时候需要和Stream&Table理论结合。

### Streams and Tables
Table: TVR是一系列的classic relation，而relation等价于table，观察一个TVR就会产生观察时刻的point-in-time relation snapshot。SQL 2011也提出了`temporal tables`的概念，可以保持不同时间版本的table，使用`AS OF SYSTEM TIME`可以查询历史的版本。

Stream: 稍微有点不同，不是每次发生变化的时候获取整个relation，而是获取导致这些snapshot的`sequence of changes`，需要定义两个假设的关键字：
- `STREAM`关键字，希望返回捕获TVR进化的event-by-event stream，可以认为是对relation使用per-record trigger
- 一个特殊的`Sys.Undo`列，用于识别召回的row

![TVR_STREAM](https://github.com/searchworld/blog/blob/master/image/TVR_STREAM.png)

`STREAM`的表示方式比较紧凑，只捕获有变化的数据；TVR table的方式比较清晰，可以看到不同时间点多的全貌。但是这两者表达方式的数据是等价的(前提是数据版本是相同的，即数据足够准确)，对应的是table/stream的两面性。但是数据足够准备一般是没法满足的，TVR table一般对应最新版本的snapshot，stream一般只保留最近一段时间的数据。这里的关键点是steam和table是同一个事情的两面性，都是编码`time-varying relations`有效的方式。

## Looking Backward: Stream and Table Biases
### The Beam Model: A Stream-Biased Approach
![Stream bias in the Beam Model approach](https://github.com/searchworld/blog/blob/master/image/stsy_0801.png)
所有逻辑transformation都是通过stream连接起来的，在beam中这些transformation是**PTransforms**，应用到**PCollections**产生新的**PCollections**，而**PCollections**在Beam中就是stream，因此Beam是stream-biased的。在Beam模型中stream是货币，而table总是特别对待的，要嘛从source和sink中抽象出来，要嘛隐藏在grouping和trigger后面。

由于Beam是在stream的维度上运行的，如果涉及到table就需要进行转换，保持table是隐藏的。这些转换包括：
- 消费table的source通常是hardcode table被trigger的方式，用户没法自己指定。
- 写table的sink通常是hardcore group input stream的方式。有可能会提供一定的灵活度。
- 对于`grouping/ungrouping operations`，Beam提供足够的灵活度在table和stream中进行转换。

在Beam中，trigger没法直接用到table上，有两个选项可用(**这里的trigger不理解？**)：
- Predeclaration of triggers，trigger在pipeline中table之前的某个点指定，是`forward-propagating`
- Post-declaration of triggers，trigger在pipeline中table之后指定，是`backward-propagating.`

在Beam模型中提供了很多方式配合实现隐藏的table，但是table在能被观察之前需要先被trigger，即使table的内容是要消费的最终数据。这是现在Beam模型的一个缺点，可以通过支持stream和table作为一等公民来实现。

### The SQL Model: A Table-Biased Approach
![Table bias with a final HAVING clause](https://github.com/searchworld/blog/blob/master/image/stsy_0804.png)

在SQL中table是一等公民，但是table并不总是显示的，隐式的table也可以存在，比如使用having语句之后就有一个table是隐式的。stream全部都是隐式的，需要转为table或者从table转回，主要有几种情况：
- Input table，在某个时间点隐式触发，产生对应时间点table的一个snapshot的一个bounded stream
- Output table，可能是query最后的grouping操作产生或者对中间结果隐式的grouping
- Grouping/ungrouping operations，这些操作只在grouping维度提供足够的灵活性。Classic SQL提供了为grouping提供了丰富的操作，但是只提供了一种类型的隐式ungrouping操作：intermediate table的上游数据已经全部到达的时候才触发。

**Materialized views**
![Table bias in materialized views](https://github.com/searchworld/blog/blob/master/image/stsy_0805.png)

和上面的区别在于`SCAN`变成`SCAN-AND-STREAM`，除了对scan时候table的snapshot产生一个bounded stream，还会对之后的变化产生unbounded stream

## Looking Forward: Toward Robust Streaming SQL
