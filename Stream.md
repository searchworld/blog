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


# Chapter 3
