# 大规模数据处理实战-01 | 为什么MapReduce会被硅谷一线公司淘汰？

![img](https://static001.geekbang.org/resource/image/54/ca/54a0178e675d0054cda83b5dc89b1dca.png)

到了 2014 年左右，Google 内部已经几乎没人写新的 MapReduce 。2016 年开始，Google 在新员工的培训中把 MapReduce 替换成了内部称为 FlumeJava（不要和 Apache Flume 混淆，是两个技术）的数据处理技术。

## **高昂的维护成本**

使用 MapReduce，你需要严格地遵循分步的 Map 和 Reduce 步骤。当你构造更为复杂的处理架构时，往往需要协调多个 Map 和多个 Reduce 任务。然而，每一步的 MapReduce 都有可能出错。为了这些异常处理，很多人开始设计自己的协调系统（orchestration）。例如，做一个状态机（state machine）协调多个 MapReduce，这大大增加了整个系统的复杂度。

想象一下这个情景，你的公司要预测美团的股价，其中一个重要特征是活跃在街头的美团外卖电动车数量，而你负责处理所有美团外卖电动车的图片。

![img](https://static001.geekbang.org/resource/image/44/c7/449ebd6c5950f5b7691d34d13a781ac7.jpg)

数据的搜集往往不全部是公司独自完成，许多公司会选择部分外包或者众包。所以在数据搜集（Data collection）部分，你至少需要 4 个 MapReduce 任务：

1. 数据导入（data ingestion）：用来把散落的照片（比如众包公司上传到网盘的照片）下载到你的存储系统。
2. 数据统一化（data normalization）：用来把不同外包公司提供过来的各式各样的照片进行格式统一。
3. 数据压缩（compression）：你需要在质量可接受的范围内保持最小的存储资源消耗 。
4. 数据备份（backup）：大规模的数据处理系统我们都需要一定的数据冗余来降低风险。

仅仅是做完数据搜集这一步，离真正的业务应用还差得远。真实的世界是如此不完美，我们需要一部分数据质量控制（quality control）流程，比如：

1. 数据时间有效性验证 （date validation）：检测上传的图片是否是你想要的日期的。
2. 照片对焦检测（focus detection）：你需要筛选掉那些因对焦不准而无法使用的照片。

最后才到你负责的重头戏——找到这些图片里的外卖电动车。而这一步因为人工的介入是最难控制时间的。你需要做 4 步：

1. 数据标注问题上传（question uploading）：上传你的标注工具，让你的标注者开始工作。
2. 标注结果下载（answer downloading）：抓取标注完的数据。
3. 标注异议整合（adjudication）：标注异议经常发生，比如一个标注者认为是美团外卖电动车，另一个标注者认为是京东快递电动车。
4. 标注结果结构化（structuralization）: 要让标注结果可用，你需要把可能非结构化的标注结果转化成你的存储系统接受的结构。

通过这个案例，我想要阐述的观点是，因为真实的商业 MapReduce 场景极端复杂，像上面这样 10 个子任务的 MapReduce 系统在硅谷一线公司司空见惯。在应用过程中，每一个 MapReduce 任务都有可能出错，都需要重试和异常处理的机制。所以，协调这些子 MapReduce 的任务往往需要和业务逻辑紧密耦合的状态机。

## **时间性能“达不到”用户的期待**

Google 曾经在 2007 年到 2012 年间做过一个对于 1PB 数据的大规模排序实验，来测试 MapReduce 的性能。从 2007 年的排序时间 12 小时，到 2012 年的排序时间缩短至 0.5 小时。即使是 Google，也花了 5 年的时间才不断优化了一个 MapReduce 流程的效率。

其中有一个重要的发现，就是他们在 MapReduce 的性能配置上花了非常多的时间。包括了缓冲大小 (buffer size），分片多少（number of shards），预抓取策略（prefetch），缓存大小（cache size）等等。

所谓的分片，是指把大规模的的数据分配给不同的机器 / 工人，流程如下图所示。

![img](https://static001.geekbang.org/resource/image/b0/38/b08b95244530aeb0171e3e35c9bfb638.png)

假如你在处理 Facebook 的所有用户数据，你选择了按照用户的年龄作为分片函数（sharding function）。我们来看看这时候会发生什么。因为用户的年龄分布不均衡（假如在 20~30 这个年龄段的 Facebook 用户最多），导致我们在下图中 worker C 上分配到的任务远大于别的机器上的任务量。

这时候就会发生掉队者问题（stragglers）。别的机器都完成了 Reduce 阶段，只有 worker C 还在工作。

![img](https://static001.geekbang.org/resource/image/63/ca/6399416524eb0dec1e292ea01b2294ca.png)

因为 MapReduce 的分片配置异常复杂，在 2008 年以后，Google 改进了 MapReduce 的分片功能，引进了动态分片技术 (dynamic sharding），大大简化了使用者对于分片的手工调整。

## **小结**

这一讲中，我们分析了两个 MapReduce 之所以被硅谷一线公司淘汰的“致命伤”：高昂的维护成本和达不到用户期待的时间性能。文中也提到了下一代数据处理技术雏型。这就是 2008 年左右在 Google 西雅图研发中心诞生的 FlumeJava，它一举解决了上面 MapReduce 的短板。另外，它还带来了一些别的优点：更好的可测试性；更好的可监控性；从 1 条数据到 1 亿条数据无缝扩展，不需要修改一行代码，等等。在后面的章节中，我们将具体展开这几点，通过深入解析 Apache Beam（FlumeJava 的开源版本），揭开 MapReduce 继任者的神秘面纱。

## **思考题**

如果你在 Facebook 负责处理例子中的用户数据，你会选择什么分片函数，来保证均匀分布的数据分片?

把年龄倒过来比如 28 岁 变成 82 来分片

使用Consistent hashing是可以很好地解决平均分配和当机器增减后重新hashing的问题。

Eventual Consistency/Strong Consistency