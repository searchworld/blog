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