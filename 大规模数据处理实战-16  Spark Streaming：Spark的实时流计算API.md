# 16 | Spark Streaming：Spark的实时流计算API

## Spark Streaming 的原理

Spark Streaming 的原理与微积分的思想很类似。

流处理的数据是一系列连续不断变化，且无边界的。我们永远无法预测下一秒的数据是什么样。Spark Streaming 用时间片拆分了无限的数据流，然后对每一个数据片用类似于批处理的方法进行处理，输出的数据也是一块一块的。如下图所示。

Spark Streaming 提供一个对于流数据的抽象 DStream。DStream 可以由来自 Apache Kafka、Flume 或者 HDFS 的流数据生成，也可以由别的 DStream 经过各种转换操作得来。

![img](https://static001.geekbang.org/resource/image/2e/a2/2e5d3fdbe0bb09a7f2cf219df1d41ca2.png)没错，底层 DStream 也是由很多个序列化的 RDD 构成，按时间片（比如一秒）切分成的每个数据单位都是一个 RDD。然后，Spark 核心引擎将对 DStream 的 Transformation 操作变为针对 Spark 中对 RDD 的 Transformation 操作，将 RDD 经过操作变成中间结果保存在内存中。

所以，Spark 是一个高度统一的平台，所有的高级 API 都有相同的性质，它们之间可以很容易地相互转化。Spark 的野心就是用这一套工具统一所有数据处理的场景。

## 滑动窗口操作

DStream 还有一些特有操作，如滑动窗口操作

StreamingContext 中最重要的参数是批处理的时间间隔，即把流数据细分成数据块的粒度。这个时间间隔决定了流处理的延迟性，所以，需要我们根据需求和资源来权衡间隔的长度。

滑动窗口操作有两个基本参数：
窗口长度（window length）：每次统计的数据的时间跨度；
滑动间隔（sliding interval）：每次统计的时间间隔。

## Spark Streaming 的优缺点

首先，Spark Streaming 的优点很明显，由于它的底层是基于 RDD 实现的，所以 RDD 的优良特性在它这里都有体现。
比如，数据容错性，如果 RDD 的某些分区丢失了，可以通过依赖信息重新计算恢复。
再比如运行速度，DStream 同样也能通过 persist() 方法将数据流存放在内存中。这样做的好处是遇到需要多次迭代计算的程序时，速度优势十分明显。
而且，Spark Streaming 是 Spark 生态的一部分。所以，它可以和 Spark 的核心引擎、Spark SQL、MLlib 等无缝衔接。换句话说，对实时处理出来的中间数据，我们可以立即在程序中无缝进行批处理、交互式查询等操作。这个特点大大增强了 Spark Streaming 的优势和功能，使得基于 Spark Streaming 的应用程序很容易扩展。
而 Spark Streaming 的主要缺点是实时计算延迟较高，一般在秒的级别。这是由于 Spark Streaming 不支持太小的批处理的时间间隔。
在第二章中，我们讲过准实时和实时系统，无疑 Spark Streaming 是一个准实时系统。别的流处理框架，如 Storm 的延迟性就好很多，可以做到毫秒级。

## 小结

Spark Streaming，作为 Spark 中的流处理组件，把连续的流数据按时间间隔划分为一个个数据块，然后对每个数据块分别进行批处理。在内部，每个数据块就是一个 RDD，所以 Spark Streaming 有 RDD 的所有优点，处理速度快，数据容错性好，支持高度并行计算。但是，它的实时延迟相比起别的流处理框架比较高。在实际工作中，我们还是要具体情况具体分析，选择正确的处理框架。

## 思考题

如果想要优化一个 Spark Streaming 程序，你会从哪些角度入手？

既然DStream底层还是RDD，那我认为针对RDD的一些优化策略对DStream也有效。比如平衡RDD分区减少数据倾斜，在tranformation链中优先使用filter/select/first减少数据量，尽量串接窄依赖函数方便实现节点间并行计算和单节点链式计算优化，join时优化分区或使用broadcast减少stage间shuffle。

另外专门针对流数据的处理，个人经验上主要是要根据业务需求微调window length和sliding interval以达到吞吐量和延时之间的一个最优平衡，时间窗口越大，一个RDD可以一次批处理的数据就越多，Spark的优势就可以发挥出来，吞吐量就上去了。而滑动间隔越大，windowed DStream在固定时间内的RDD就越少，系统的任务队列里同时需要处理的计算当然就越少，不过这两个调整都会加大数据更新延迟和牺牲数据实时性，所以说要根据业务真实需求谨慎调整。

不过个人理解RDD里面用来避免重复计算的cache和persist无法用来减少窗口滑动产生的重复计算，因为窗口每滑动一次，都产生一个新的RDD，而persist只针对其中某个RDD进行缓存，在RDD这种low level api里面，应该是无法知道下个窗口中的RDD和现在的RDD到底有多少数据是重叠的。