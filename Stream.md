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
providing low-latency, inaccurate, or speculative results，通常和batch系统一起，提供最终一致性的结果，接[Lambda架构](http://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html)，这个架构存在的[问题](https://oreil.ly/2LSEdqz)，需要维护两套系统，最后可能需要维护最终状态。

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

不足：如果这些事件有相应的事件时间，那processing-time window如果要反应这些事件的真实情况就要求事件按照event time的顺序到达。

### Windowing by event time
可以创建固定窗口和动态窗口(比如Session)。event-time窗口要比实际的窗口长，有两个缺点：
- Buffering，窗口更长，需要buffer更多数据。持久化存储比较便宜，因此这个问题不大，另外有些场景(比如sum)可以增量进行，不需要缓存所有数据
- Completeness，对于许多类型的输入可以通过watermark来进行启发式估计。对于需要绝对准确的场景(比如账单)，只能定义什么时候对窗口求值，并随着时间推移对这个结果进行refine


# Chapter 2. The What, Where,When, and How of Data Processing
Chapter1说明了两个概念：event-time vs processing-time 和 window。新增三个概念：
- Triggers，声明window的输出和外部信号相比什么时候materialized的机制，提供了什么时候emit输出的灵活性，可以看做是流控机制用来控制结果要在什么时候materialized。另一个角度可以看做是相机的快门，控制什么时候拍照(结果输出)
- Watermarks
- Accumulation

