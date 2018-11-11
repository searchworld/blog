# Chapter 1. Stream 101

# Background
## Terminology
- Stream system
  - cardinality
    - Bounded data
    - Unbounded data
  - constitution
    - Table
    - Stream

## Limitation
providing low-latency, inaccurate, or speculative results，通常和batch系统一起，提供最终一致性的结果，即[Lambda架构](http://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html)，这个架构存在的[问题](https://www.oreilly.com/ideas/questioning-the-lambda-architecture)，需要维护两套系统，最后可能需要维护最终状态。

stream system要打败batch system只需要在两方面上下功夫：
- correctness，这就和batch system等价，其核心是consistent storage，因此需要对状态不停做checkpoint且在机器失败的时候仍然保持一致性。strong consistent对于exact-once processing是必须的。

  达到一致性的三篇论文[MillWheel](http://bit.ly/2Muob70),[spark streaming](http://bit.ly/2Mrq8Be),[Flink snapshot](http://bit.ly/2t4DGK0)
- Tools for reasoning about time，这里可以超越batch system。这是处理unbounded, unordered data of varying eventtime
skew的关键。

## Event time vs Processing time
- Event time，事件真正发生的时间
- Processing time，系统观察到event的时间

如果不需要关注event time，系统会简单很多。

在一个理想的世界里，这两个时间应该是一样的，即事件发生之后立刻被处理。现实中这两个时间差别可能很大，event time和Processing time的关系不是确定的，因此无法通过分析Processing time得到event time，基于processing time的window得到的结果不是完整的。

不要尝试将unbounded stream转成有限的最终可以得到完整结果的batch，completeness应该作为一些特殊场景的优化，而不是所有场景都需要的语义。


# Data Processing Patterns

## Bounded Data

## Unbounded Data: Batch
### Fixed windows
在流处理前unbounded data已经收集成有限的固定窗口大小的有限数据，再进入batch engine

### Session
典型的Session可以是被超过一段inactive时间而终止的用户活动区间。一个Session可能会跨batch。

## Unbounded Data: Streaming

### Time-agnostic
用于时间无关的场景，所有逻辑都是数据驱动的

- Filtering，比如过滤掉某个域名的访问日志，这是时间无关的
- Inner joins，当对两个unbounded stream做join的时候只关心来自两个source的元素都已经到达，逻辑中没有时间的维度。outer join由于需要知道另一方的数据是否已经完整，本质上还是需要引入window的概念

### Approximation algorithms
比如approximate Top-N和streaming k-means，输入一个unbounded源，输出一个近似的结果。这种算法只有少数几个，而且很复杂。这些算法的设计中通常都会有时间元素，而且通常是processing time

### Windowing
按时间边界分为有限的块。
- Fixed windows，固定时间长度
- Sliding windows，fixed-length & fixed-period，如果period小于length，则window之间重叠；如果两者相等，则等价于固定窗口；如果period大于length，则类似采样
- Sessions，动态窗口，当inactive超过timeout的时候，之前的事件就切出一个Session，可以用于分析用户行为

## 两种不同的时间窗口

### Windowing by processing time
历史上这种window更常见，系统将进来的数据缓存起来，直到超过一段processing time

优势：
- 简单，只要做一下buffer，当时间到达的时候关闭窗口
- 窗口完整性，不需要考虑延迟到达的数据
- 只关注事件被系统观察到的状态，比如监控每秒的请求数

不足：如果这些事件有相应的事件时间，那processing-time window如果要反映这些事件的真实情况就要求事件按照event time的顺序到达。

### Windowing by event time
可以创建固定窗口和动态窗口(比如Session)。event-time窗口要比实际的窗口长，有两个缺点：
- Buffering，窗口更长，需要buffer更多数据。持久化存储比较便宜，因此这个问题不大，另外有些场景(比如sum)可以增量进行，不需要缓存所有数据
- Completeness，对于许多类型的输入可以通过watermark来进行启发式估计。对于需要绝对准确的场景(比如账单)，只能定义什么时候对窗口求值，并随着时间推移对这个结果进行refine


# Chapter 2. The What, Where,When, and How of Data Processing
## Roadmap
Chapter1说明了两个概念：event-time vs processing-time 和 window。新增三个概念：
- Triggers，声明相对于外部信号window的输出什么时候materialized的机制，提供了什么时候emit输出的灵活性，可以看做是流控机制用来控制结果要在什么时候materialized。另一个角度可以看做是相机的快门，控制什么时候拍照(结果输出)

同时trigger也让一个window的输出被观察多次称为可能，这可以用来对结果进行refine。
- Watermarks，用event-time来描述输入完整性的概念，时间为`X`的watermark表示：所有event-time小于`X`的输入数据已经被观察到。因此可以用来测量unbounded数据的进度
- Accumulation，一个accumulation mode用来指定同一个窗口多个结果直接的关系，这些结果可以没有任何关系，也可以有重叠

对于unbounded data processing都很重要的问题：
- **What** results are calculated? 由pipeline中的transformation类型回答，可以是sum或者机器学习，batch processing也要回答这个问题
- **Where** in event time are results calculated? 在pipeline中使用event-time window来解决，比如fix/slided/session等
- **When** in processing time are results materialized? 使用trigger和watermark解决，
- **How** do refinements of results relate? 使用accumulation回答，可以是discarding或者accumulated等

## Batch Foundations: What and Where
### What: Transformations
### Where: Windowing

## Going Streaming: When and How
### When: Triggers
trigger定义window的输出在process-time(也可以是其它的time domain，比如使用event-time衡量的watermark)时间点产生。window的每个输出称为window的一个**pane**。

概念上通常有两种有用的trigger类型，实际的应用基本使用这两种的一种或者两者结合。
- Repeated update triggers。周期性为window产生一个更新过的pane，这些更新可以是每条记录之后，也可以是一个processing-time delay。周期的选择主要是latency和cost的权衡
- Completeness triggers。只有在认为window的输入对于某个阈值来说是完整的才会产生一个pane。这有点类似batch processing，主要区别是这里的完整性是建立在window的上下文上，而不是所有input

前者比较常见，实现和理解起来也比较简单，类似数据库中的materialized views；后者比较类似batch processing，提供了missing data和late data这样的工具。

per-record trigger可以实时得到结果，对于数据量比较大的场景消耗比较大。有两种processing-time delay:
- `aligned delays`，将processing-time分成对齐的region，spark streaming中的micro batch就是这种类型，其优势是可预测性，在同一时间点得到结果；劣势是所有更新同时发生，导致突发性的workload
- `unaligned delays`，delay和窗口中观察到的数据有关，workload的分布比较均匀，对于大数据量是个比较好的选择

### When: Watermarks
event-time域输入完整性的时间概念，系统用来衡量相对record中event-time的进度和完整性。有两种类型：
- Perfect watermarks，用于知道输入数据的所有信息的场景，不会有late data的场景，所有数据都是on time，对于大部分输入这是不太现实的
- Heuristic watermarks，根据输入的信息尽可能得到准确的进度评估，在许多场景下可以非常准确，但是仍然可能存在late data。

watermark构成了`completeness triggers`的基础，只有系统materializes一个window的输出，我们才认为这个window的输入是完整的，这对于需要知道`missing data`的场景很重要，比如outer-join。

watermark也有不足的地方：
- Too slow，当watermark由于数据未处理产生delay，会直接反应到输出的delay。如果对latency比较敏感，watermark不太适合
- Too fast，如果heuristic watermark错误的advance，会产生很多的late data

将`Repeated update trigger`和`Completeness trigger`结合可以解决上面太快和太慢的问题

### When: Early/On-Time/Late Triggers FTW!
`early/on-time/late trigger`将pane分成几个部分：
- 0个或者多个`early panes`，watermark还没到达，`repeated update trigger`周期触发产生的结果，解决上面too slow的问题
- 一个`on-time pane`，`completeness/watermark trigger`在watermark经过window右侧的时候产生的结果，提供了断言输入已经完成，可以解决`missing data`的问题
- 一个或者多个`late panes`，watermark已经经过window，`repeated update trigger`在`late data`到来的时候触发，对于perfect watermark总是0个。

### When: Allowed Lateness (i.e., Garbage Collection)
使用`early/on-time/late trigger`需要一直保留所有的state，因为不知道late data什么时候会到来，这对于unbounded data是不太现实的，因此需要限制window的生命周期，最简单的方式是给出late data的范围，当watermark超过这个范围则对应的窗口关闭，超过这个范围的数据认为是不需要处理的，直接丢弃。这个时间可以有两种指定方式：
- watermark超过窗口右侧10分钟，和event-time绑定，不会出现window没有处理late data的机会
- processing-time超过右侧窗口10分钟(spark使用的方式)，如果机器crash则有可能出现late data没有被处理

### How: Accumulation
trigger可能产生多个pane，这就存在一个问题：要怎么对多个pane的结果进行处理。accumulation有三种模式：
- Discarding，当一个新的pane产生的时候，之前存储的state被丢弃，即pane之间没有什么联系，用于下游的消费者自己进行accumulation的场景，比如每次只产生一个delta，下游自己进行sum
- Accumulating，每个pane建立在之前pane的基础上，原来的state继续保留，未来的输入累加到这个state上。用于后面的结果可以覆盖之前结果的场景，比如结果存储在hbase中
- Accumulating and retracting，类似accumulation模式，但是每次产生一个新的pane，会对之前的pane产生一个retraction，表达的语义是：我之前告诉你结果是X，但是这个结果是错误的，删除我上次给你的X，用这次的Y替代。下面两种场景下很有用：
  - 下游的consumer对数据按照不同的维度进行regroup，新的结果的key可能和之前的key完全不同，不能简单的覆盖
  - 当使用动态窗口(比如session window)，由于窗口合并，新的值可能会替换多个之前的窗口。这时候很难直接根据新的窗口得到替换的旧窗口，需要显示给定retraction

这三种accumulation的方式的消耗从小到大。


# Chapter 3 Watermarks
一个前提假设：每条消息都关联一个逻辑上的事件时间戳。这个假设是合理的，因为连续到达的unbounded数据说明有输入输入的持续产生。

消息被pipeline ingest，然后process，最后标志为complete；每条消息的状态可能是`in-flight`，表示已经被接收，但是还没有处理完成，或者`completed`，表示不需要对这条消息再做任何处理。随着时间的推移，越来越多的消息加入到`in-flight`，越来越多的消息从`in-flight`转为`completed`，`in-flight`最左侧的点对应pipeline中未处理的最老的数据，可以用这个值来定义`watermark`: 还没有完成的最老的work对应的单调递增的时间戳。这个定义有两个属性：
- Completeness，如果watermark已经超过时间T，单调属性保证T之前的on-time的消息不需要再被处理，因此可以发出T之前的聚合结果。即我们知道什么时候可以正确关闭一个窗口
- Visibility，如果消息因为某种原因stuck在pipeline的某个地方，watermark不能前进，我们可以检查阻止watermark前进消息来找到问题的根源

## Source Watermark Creation
perfect和heuristic watermark的区别：前者保证计算所有数据，后者可以有late data。一旦创建之后就保持不变，具体是什么类型主要依赖上游的source的属性。

## Watermark Propagation
概念上可以认为watermark随着数据处理超过之后从系统流过。有多个stage组成的pipeline每个stage会追踪自己的watermark，其值是当前stage之前的input和stages的的一个函数。因此越往后的stage其watermark越晚。

在stage的边界上定义watermark有利于理解pipeline中每个stage的相对进度，还可以及时独立分发及时的结果。按边界定义有两种watermark:
- `input watermark`，捕获当前stage的所有upstream的进度，对于source，input watermark是一个source相关的创建watermark的方法；对于nosource stage，input watermark是所有上游output watermark的最小值
- `output watermark`，反应stage本身的进度，定义为stage的input watermark和stage中`all nonlate data active messages`的event-time的最小值。这里的`active`依赖这个stage执行的动作和系统的实现，通常包括准备做Aggregation的缓存数据，pending output data等

`input watermark` - `output watermark`可以得到这个stage引入的event-time latency。比如一个执行10s window聚合的窗口的stage相对输出会有10s或者更大的延迟，对输入和输出watermark的定义提供了整个pipeline的watermark的递归关系。

每个stage中的处理不是一个整体，可以分成概念上的几个component组成的flow，每个组件都对output watermark有贡献。component的具体属性依赖stage的内容和系统实现，概念上component的实现是作为一个buffer，active message可以保留到某个操作完成。然后可能会将数据写入state，用于后面的延迟聚合，延迟聚合之后可能会将结果写入output buffer，供downstream stage消费。我们可以使用每个buffer自己的watermark来跟踪每个buffer，这样可以方便的看到watermark在哪里stuck，同时stage中所有buffer的最小的watermark构成了这个stage的watermark。因此output watermark可以是下面某一个的最小值：
- Per-source watermark—for each sending stage.
- Per-external input watermark—for sources external to the pipeline
- Per-state component watermark—for each type of state that can be written
- Per-output buf er watermark—for each receiving stage

### Watermark Propagation and Output Timestamps
watermark不允许向后移动，因此一个window有效的output timestamp从第一个nonlate元素开始到无限。实践中一般取下面三种作为output timestamp:
- End of the window，如果要让output timestamp可以代表window bound，这是唯一的安全选择，同时也是smoothest watermark progression。
- Timestamp of first nonlate element，如果要让watermark尽量保守，这是一个好选择。带来的问题是 watermark progress更容易受阻
- Timestamp of a specific element，有效场景下随机选一个元素的timestamp是个好选择，比如对一个点击流和一个查询流做join，有些人可能会觉得点击流的时间更重要，有些人可能会认为查询流的更重要。

### The Tricky Case of Overlapping Windows
对于sliding window，可能会出现结果被delay的情况。**没太理解？**

## Percentile Watermarks
除了使用最小的event time来测量watermark，还可以使用消息分布的百分比还测量，表达的含义是：已经处理了具有较早timestamp的所有事件的这个百分比数量。对于几乎正确就可以满足的场景，`percentile watermark`可以前进的更快更顺滑，因为丢掉了watermark分布的长尾。

## Processing-Time Watermarks
使用event-time watermark不足以区分出一个系统是在处理old data还是delay了。换句话说，一个系统在快速处理一个小时前的数据且没有delay；一个系统在处理实时数据，但是延时了一个小时，通过event-time watermark是无法区分开的。

processing-time watermark的定义和event-time watermark一样，只是后者使用尚未完成的最老的工作的event-time作为watermark，前者使用尚未完成的最老的operation的processing-time作为watermark。processing-time watermark delay的原因可能是一条消息延迟从一个stage传递到另一个stage，也可能是读取状态或者外部数据的I/O延时，或者处理异常。通常需要管理员介入处理。因此processing-time watermark是一个区分data latency和system latency的工具，同时也可以用来垃圾回收临时状态。

## Case Studies
`Google Cloud Dataflow`中的watermark实现是通过一个集中式的`watermark aggregator agent`来实现。这里面需要保证多个worker不会并发去修改state，优势是：
- watermark来自一个source，方便调试监控
- source watermark创建，比如有些需要全局信息的source

Flink使用in-band的方式，即watermark和data stream 被同步in-band发送出去。这种方式的优势是：
- 低延时
- 没有单点失败
- 内置的可扩展性


# Chapter 4. Advanced Windowing

## When/Where: Processing-Time Windows
适合类似使用监控的场景，比如web service QPS。在我们的系统中，作为一等公民的window都是基于event-time的，要得到processing-time window有两种方式：
- Trigger，忽略event-time(比如使用global window)，使用Trigger提供processing-time的window的快照
- Ingest time，在数据到达的时候赋予ingest time，然后使用正常的event-time window。spark 1.x就是使用这种方式

这两者基本等价，但是对于多stage的场景可能会**细微差别(?)**：Trigger方式在每个stage去划分window，同样的数据可能进入不同的window；ingest-time版本不会出现这种情况

## Where: Session Windows
有两个特征：
- data-driven window，window的大小和位置与输入有关，
- unaligned window

先对每个元素求window，然后做merge

## Where: Custom Windowing
一个自定义的window主要包括两部分：
- window assignment，将每个元素放入一个初始化的window
- window merging(可选的)，允许在group的时候对window做merge，让window随着时间进化称为可能，比如上面的session window

### Variations on Fixed Windows
普通的fixed window很简单，只需要window assignment这一步，根据时间戳、window大小和offset将元素放入合适的窗口。这种window是aligned的，即所有窗口会在同一个时间点触发，可能造成很高的负载。

fixedwindow有两个变种：
- Unaligned fixed windows。相同key是align的，不同的key可以不对其，这样可以把负载分散开
- Per-element/key fixed windows，每个key可以指定自己的window大小(在meta信息中指定)

### Variations on Session Windows
典型的session window实现如下：
- assignment，每个元素最初被放入一个proto-session window，这个窗口的从元素的开始时间开始延伸到gap duration
- merging，在group的时候所有合适的窗口会进行排序，任何重叠的窗口会合并到一起

session window的一个变种：Bounded sessions，即session的大小是固定的，不允许超过一定大小(可能是时间或者元素，或者其他维度)

