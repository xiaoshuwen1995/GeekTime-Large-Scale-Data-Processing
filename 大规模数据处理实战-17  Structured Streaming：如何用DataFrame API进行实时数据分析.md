# 17 | Structured Streaming：如何用DataFrame API进行实时数据分析?

2016 年，Spark  2.0 版本推出了结构化流数据处理的模块 Structured Streaming。Structured Streaming 是基于 Spark SQL 引擎实现的，我们可以像批处理静态数据那样去处理流数据。

随着流数据的持续输入，Spark SQL 引擎会帮助我们持续地处理新数据，并且更新计算结果。

## Structured Streaming 模型

流数据处理最基本的问题，就是如何对不断更新的无边界数据建模。之前讲的 Spark Streaming 就是把流数据按一定的时间间隔分割成许多个小的数据块，进行批处理。

在 Structured Streaming 的模型中，我们要把数据看成一个无边界的关系型的数据表。每一个数据都是表中的一行，不断会有新的数据行被添加到表里来。

我们可以对这个表做类似批处理的查询，Spark 也会帮我们不断对新加入的数据进行处理并更新计算结果。

![img](https://static001.geekbang.org/resource/image/bb/37/bb1845be9f34ef7d232a509f90ae0337.jpg)



Structured Streaming 的三种输出模式：

- 完全模式（Complete Mode）：整个更新过的输出表都被写入外部存储；
- 附加模式（Append Mode）：上一次触发之后，新增加的行才会被写入外部存储。如果老数据有改动则不适合这个模式；
- 更新模式（Update Mode）：上一次触发之后，被更新的行才会被写入外部存储。

Structured Streaming 并不会完全存储输入数据，Structured Streaming 只会存储更新输出表所需要的信息。每个时间间隔它都会读取最新的输入，进行处理并更新输出表，然后把这次的输入删除。

Structured Streaming 的模型在根据事件时间（Event Time）处理数据时十分方便。

由于每个数据都是输入数据表中的一行，那么事件时间就是行中的一列。依靠 DataFrame API 提供的类似于 SQL 的接口，我们可以很方便地执行基于时间窗口的查询。

## Structured Streaming 与 Spark Streaming 对比

1. 简易度和性能

 DataFrame API 是相对高 level 的 API，可以方便开发者用一套统一的方案去处理批处理和流处理，不用去关心具体的执行细节。

而且DataFrame API 是在 Spark SQL 的引擎上执行的，Spark SQL 有执行计划优化和内存管理等，所以 Structured Streaming 的应用程序性能很好。

2. 实时性

虽然 Structured Streaming 用的是类似的微批处理思想，每过一个时间间隔就去拿来最新的数据加入到输入数据表中并更新结果，但是相比起 Spark Streaming 来说，它更像是实时处理，能做到用更小的时间间隔，最小延迟在 100 毫秒左右。

Structured Streaming 对基于事件时间的处理有很好的支持。

由于 Spark Streaming 是把数据按接收到的时间切分成一个个 RDD 来进行批处理，所以它很难基于数据本身的产生时间来进行处理。如果某个数据的处理时间和事件时间不一致的话，就容易出问题。比如统计每秒的词语数量，有的数据先产生，但是在下一个时间间隔才被处理，这样几乎不可能输出正确的结果。

## 思考题

在基于事件时间的窗口操作中，Structured Streaming 是怎样处理晚到达的数据，并且返回正确结果的呢？比如，在每十分钟统计词频的例子中，一个词语在 1:09 被生成，在 1:11 被处理，程序在 1:10 和 1:20 都输出了对应的结果，在 1:20 输出时为什么可以把这个词语统计在内？这样的机制有没有限制？

1、Structured Streaming是基于事件事件处理，而不是处理事件，所以，延迟接收的数据，是能被统计到对应的事件时间窗口的

2、设定数据延迟的窗口时间阈值，通过判断阈值来决定延迟数据是否需要纳入统计；这个阈值的设定可以避免大量数据的延迟导致的性能问题