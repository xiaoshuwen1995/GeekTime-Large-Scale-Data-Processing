# 21 | 深入对比Spark与Flink：帮你系统设计两开花

 Spark 的主要优点，比如用内存运算来提高性能；提供很多 High-level API；开发者无需用 map 和 reduce 两个操作实现复杂逻辑；支持流处理等等。

RDD 是整个 Spark 的核心概念，所有的新 API 在底层都是基于 RDD 实现的。但是 RDD 是否就是完美无缺的呢？显然不是，它还是很底层，不方便开发者使用，而且用 RDD API 写的应用程序需要大量的人工调优来提高性能。

Spark SQL 提供的 DataFrame/DataSet API 就解决了这个问题，它提供类似 SQL 的查询接口，把数据看成关系型数据库的表，提升了熟悉关系型数据库的开发者的工作效率。这部分内容都是专注于数据的批处理。

但 Spark 的流处理是基于所谓微批处理（Micro-batch processing）的思想，即它把流处理看作是批处理的一种特殊形式，每次接收到一个时间间隔的数据才会去处理，所以天生很难在实时性上有所提升。

Apache Flink 流处理框架，采用了基于操作符（Operator）的连续流模型，可以做到微秒级别的延迟。每当有一条数据输入就立刻处理，不做等待。

## Flink 核心模型简介

Flink 中最核心的数据结构是 Stream，它代表一个运行在多个分区上的并行流。在 Stream 上同样可以进行各种转换操作（Transformation）。

与 Spark 的 RDD 不同的是，Stream 代表一个数据流而不是静态数据的集合，所以它包含的数据是随着时间增长而变化的。

而且 Stream 上的转换操作都是逐条进行的，即每当有新的数据进来，整个流程都会被执行并更新结果。

这样的基本处理模式决定了 Flink 会比 Spark Streaming 有更低的流处理延迟性。

![img](https://static001.geekbang.org/resource/image/c4/b5/c49f4155d91c58050d8c7a2896bbc9b5.jpg)

在图中 Streaming Dataflow 包括 Stream 和 Operator（操作符）。转换操作符把一个或多个 Stream 转换成多个 Stream。每个 Dataflow 都有一个输入数据源（Source）和输出数据源（Sink）。与 Spark 的 RDD 转换图类似，Streaming Dataflow 也会被组合成一个有向无环图去执行。

在 Flink 中，程序天生是并行和分布式的。一个 Stream 可以包含多个分区（Stream Partitions），一个操作符可以被分成多个操作符子任务，每一个子任务是在不同的线程或者不同的机器节点中独立执行的。如下图所示：

![img](https://static001.geekbang.org/resource/image/e9/58/e90778ee8f3cf092d80b73dca59a8658.jpg)

从上图你可以看出，Stream 在操作符之间传输数据的形式有两种：一对一和重新分布。

- 一对一（One-to-one）：Stream 维护着分区以及元素的顺序，比如上图从输入数据源到 map 间。这意味着 map 操作符的子任务处理的数据和输入数据源的子任务生产的元素的数据相同。你有没有发现，它与 RDD 的窄依赖类似。
- 重新分布（Redistributing）：Stream 中数据的分区会发生改变，比如上图中 map 与 keyBy 之间。操作符的每一个子任务把数据发送到不同的目标子任务。

## Flink 的架构

![img](https://static001.geekbang.org/resource/image/72/8a/7279dcfede45e83e1f8a9ff28cca178a.png)

我们可以看到，这个架构和第 12 讲中介绍的 Spark 架构比较类似，都分为四层：存储层、部署层、核心处理引擎、high-level 的 API 和库。

Flink 提供的两个核心 API 就是 DataSet API 和 DataStream API。你没看错，名字和 Spark 的 DataSet、DataFrame 非常相似。顾名思义，DataSet 代表有界的数据集，而 DataStream 代表流数据。所以，DataSet API 是用来做批处理的，而 DataStream API 是做流处理的。

也许你会问，Flink 这样基于流的模型是怎样支持批处理的？在内部，DataSet 其实也用 Stream 表示，静态的有界数据也可以被看作是特殊的流数据，而且 DataSet 与 DataStream 可以无缝切换。所以，Flink 的核心是 DataStream。

在这个例子中，我们首先创建了一个 Splitter 类，来把输入的句子拆分成（词语，1）的对。在主程序中用 StreamExecutionEnvironment 创建 DataStream，来接收本地 Web Socket 的文本流，并进行了 4 步操作。用 flatMap 把输入文本拆分成（词语，1）的对；用 keyBy 把相同的词语分配到相同的分区；设好 5 秒的时间窗口；对词语的出现频率用 sum 求和。

## Flink 和 Spark 对比

首先，这两个数据处理框架有很多相同点。

1. 都基于内存计算；
2. 都有统一的批处理和流处理 API，都支持类似 SQL 的编程接口；
3. 都支持很多相同的转换操作，编程都是用类似于 Scala Collection API 的函数式编程模式；
4. 都有完善的错误恢复机制；
5. 都支持 Exactly once 的语义一致性。

当然，它们的不同点也是相当明显，我们可以从 4 个不同的角度来看。

1. 从流处理的角度来讲，Spark 基于微批量处理，把流数据看成是一个个小的批处理数据块分别处理，所以延迟性只能做到秒级。而 Flink 基于每个事件处理，每当有新的数据输入都会立刻处理，是真正的流式计算，支持毫秒级计算。由于相同的原因，Spark 只支持基于时间的窗口操作（处理时间或者事件时间），而 Flink 支持的窗口操作则非常灵活，不仅支持时间窗口，还支持基于数据本身的窗口，开发者可以自由定义想要的窗口操作。
2. 从 SQL 功能的角度来讲，Spark 和 Flink 分别提供 SparkSQL 和 Table API 提供 SQL 交互支持。两者相比较，Spark 对 SQL 支持更好，相应的优化、扩展和性能更好，而 Flink 在 SQL 支持方面还有很大提升空间。
3. 从迭代计算的角度来讲，Spark 对机器学习的支持很好，因为可以在内存中缓存中间计算结果来加速机器学习算法的运行。但是大部分机器学习算法其实是一个有环的数据流，在 Spark 中，却是用无环图来表示。而 Flink 支持在运行时间中的有环数据流，从而可以更有效的对机器学习算法进行运算。
4. 从相应的生态系统角度来讲，Spark 的社区无疑更加活跃。Spark 可以说有着 Apache 旗下最多的开源贡献者，而且有很多不同的库来用在不同场景。而 Flink 由于较新，现阶段的开源社区不如 Spark 活跃，各种库的功能也不如 Spark 全面。但是 Flink 还在不断发展，各种功能也在逐渐完善。

## 小结

以下场景可以选择 Spark，一站式解决这些问题：

1. 数据量非常大而且逻辑复杂的批数据处理，并且对计算效率有较高要求（比如用大数据分析来构建推荐系统进行个性化推荐、广告定点投放等）；

2. 基于历史数据的交互式查询，要求响应较快；

3. 基于实时数据流的数据处理，延迟性要求在在数百毫秒到数秒之间。

由于 Flink 是为了提升流处理而创建的平台，所以它适用于各种需要非常低延迟（微秒到毫秒级）的实时数据处理场景，比如实时日志报表分析。

窗口是流数据处理中最重要的概念之一，窗口定义了如何把无边界数据划分为一个个有限的数据集。基于事件时间的窗口只是窗口的一种，它是按照事件时间的先后顺序来划分数据，比如说1:00-1:10是一个集合，1:10-1:20又是一个集合。

但是窗口并不都是基于时间的。比如说我们可以按数据的个数来划分，每接受到10个数据就是一个集合，这就是Count-based Window（基于数量的窗口）。Flink对于窗口的支持远比Spark要好，这是它相比Spark最大的优点之一。它不仅支持基于时间的窗口（处理时间、事件时间和摄入时间），还支持基于数据数量的窗口。

此外，在窗口的形式上，Flink支持滚动窗口（Tumbling Window）、滑动窗口（Sliding Window）、全局窗口（Global Window）和会话窗口（Session Windows）。