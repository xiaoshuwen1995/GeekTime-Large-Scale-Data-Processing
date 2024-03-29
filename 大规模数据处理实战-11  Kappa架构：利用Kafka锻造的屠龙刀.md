# 11 | Kappa架构：利用Kafka锻造的屠龙刀

Lambda 架构结合了批处理和流处理的架构思想，将进入系统的大规模数据同时送入这两套架构层中，分别是批处理层（Batch Layer）和速度层（Speed Layer），同时产生两套数据结果并存入服务层。批处理层有着很好的容错性，同时也因为保存着所有的历史记录，使产生的数据集具有很好的准确性。速度层可以及时地处理流入的数据，因此具有低延迟性。最终服务层将这两套数据结合，并生成一个完整的数据视图提供给用户。

## Lambda 架构的不足

使用 Lambda 架构时，架构师需要维护两个复杂的分布式系统，并且保证他们逻辑上产生相同的结果输出到服务层中。维护 Lambda 架构的复杂性在于我们要同时维护两套系统架构：批处理层和速度层。我们已经说过了，在架构中加入批处理层是因为从批处理层得到的结果具有高准确性，而加入速度层是因为它在处理大规模数据时具有低延时性。

那我们能不能改进其中某一层的架构，让它具有另外一层架构的特性呢？例如，改进批处理层的系统让它具有更低的延时性，又或者是改进速度层的系统，让它产生的数据视图更具准确性和更加接近历史数据呢？

## Kappa 架构

像 Apache Kafka 这样的流处理平台是具有永久保存数据日志的功能的。通过平台的这一特性，我们可以重新处理部署于速度层架构中的历史数据。

第一步，部署 Apache Kafka，并设置数据日志的保留期（Retention Period）。这里的保留期指的是你希望能够重新处理的历史数据的时间区间。例如，如果你希望重新处理最多一年的历史数据，那就可以把 Apache Kafka 中的保留期设置为 365 天。如果你希望能够处理所有的历史数据，那就可以把 Apache Kafka 中的保留期设置为“永久（Forever）”。

第二步，如果我们需要改进现有的逻辑算法，那就表示我们需要对历史数据进行重新处理。我们需要做的就是重新启动一个 Apache Kafka 作业实例（Instance）。这个作业实例将重头开始，重新计算保留好的历史数据，并将结果输出到一个新的数据视图中。我们知道 Apache Kafka 的底层是使用 Log Offset 来判断现在已经处理到哪个数据块了，所以只需要将 Log Offset 设置为 0，新的作业实例就会重头开始处理历史数据。

第三步，当这个新的数据视图处理过的数据进度赶上了旧的数据视图时，我们的应用便可以切换到从新的数据视图中读取。

第四步，停止旧版本的作业实例，并删除旧的数据视图。

![img](https://static001.geekbang.org/resource/image/f4/ff/f45975d67c2bf9640e6361c8c23727ff.png)

与 Lambda 架构不同的是，Kappa 架构去掉了批处理层这一体系结构，而只保留了速度层。你只需要在业务逻辑改变又或者是代码更改的时候进行数据的重新处理。

## 《纽约时报》内容管理系统架构实例

![img](https://static001.geekbang.org/resource/image/73/61/73877382d7b1bfc01bbcd9951ec7ca61.png)

首先，Kappa 架构在系统调度这个层面上统一了开发接口。你可以看到，中间的 Kappa 架构系统规范好了输入数据和输出数据的格式之后，任何需要传送到应用端的数据都必须按照这个接口输入给 Kappa 架构系统。而所有的应用端客户都只需要按照 Kappa 架构系统定义好的输出格式接收传输过来的数据。这样就**解决了 API 规范化**的问题。

因为 Apache Kafka 是可以实时推送消息数据的，这样一来，任何传输进中间 Kappa 架构的数据都会被实时推送到接收消息的客户端中。这样就避免了在应用层面上做定期轮询，从而**减少了延时**。而对于重新访问或者处理发布过的新闻内容这一问题，还记得我之前和你讲述过的 Kafka 特性吗？只需要设置 Log Offset 为 0 就可以重新读取所有内容了。

## Kappa 架构也是有着它自身的不足的

因为 Kappa 架构只保留了速度层而缺少批处理层，在速度层上处理大规模数据可能会有数据更新出错的情况发生，这就需要我们花费更多的时间在处理这些错误异常上面。

还有一点，Kappa 架构的批处理和流处理都放在了速度层上，这导致了这种架构是使用同一套代码来处理算法逻辑的。所以 Kappa 架构并不适用于批处理和流处理代码逻辑不一致的场景。

## 小结

如果你所面对的业务逻辑是设计一种稳健的机器学习模型来预测即将发生的事情，那么你应该优先考虑使用 Lambda 架构，因为它拥有批处理层和速度层来确保更少的错误。

如果你所面对的业务逻辑是希望实时性比较高，而且客户端又是根据运行时发生的实时事件来做出回应的，那么你就应该优先考虑使用 Kappa 架构。

## 思考题

在学习完 Lambda 架构和 Kappa 架构之后，你能说出 Kappa 架构相对 Lambda 架构的优势吗？

参考答案：

1.kappa架构使用更少的技术栈，实时和历史部分都是同一套技术栈。lambda架构为了解决历史部分和实时部分可能会使用不同的技术栈。

2.kappa架构使用了统一的处理逻辑。而lambda架构分别为历史和实时部分使用了两套逻辑。一旦需求变更，两套逻辑都要同时变更。

3.kappa架构具有流式处理的特点和优点。比如可以具有多个订阅者，比如具有更高的吞吐量。