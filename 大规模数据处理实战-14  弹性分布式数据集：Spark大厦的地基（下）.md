# 14 | 弹性分布式数据集：Spark大厦的地基（下）

这一讲是RDD 结构中其他的几个知识点：检查点（Checkpoint）、存储级别（ Storage Level）和迭代函数（Iterator）。

![img](https://static001.geekbang.org/resource/image/8c/1c/8cae25f4d16a34be77fd3e84133d6a1c.png)

如果一个 RDD 的依赖链比较长，而且中间又有多个 RDD 出现故障的话，进行恢复可能会非常耗费时间和计算资源。

而检查点（Checkpoint）的引入，就是为了优化数据恢复。很多数据库系统都有检查点机制，在连续的 transaction 列表中记录某几个 transaction 后数据的内容，从而加快错误恢复。

在计算过程中，对于一些计算过程比较耗时的 RDD，可以将它缓存至硬盘或 HDFS 中，标记这个 RDD 有被检查点处理过，并且清空它的所有依赖关系。

同时，给它新建一个依赖于 CheckpointRDD 的依赖关系，CheckpointRDD 可以用来从硬盘中读取 RDD 和生成新的分区信息。

这样，当某个子 RDD 需要错误恢复时回溯，如果它被检查点记录过，就可以直接去硬盘中读取这个 RDD，而无需再向前回溯计算。

迭代函数会首先判断缓存中是否有想要计算的 RDD，如果有就直接读取，如果没有就查找想要计算的 RDD 是否被检查点处理过。如果有，就直接读取，如果没有，就调用计算函数向上递归，查找父 RDD 进行计算。

## RDD 的转换操作

RDD 的数据操作分为两种：转换（Transformation）和动作（Action）。转换是用来把一个 RDD 转换成另一个 RDD，而动作则是通过计算返回一个结果。

 map、filter、groupByKey 等都属于转换操作。

 map 与 MapReduce中的一样，它把 RDD 中通过一个函数映射成一个新的 RDD，任何原 RDD 中的元素在新 RDD 中都有且只有一个元素与之对应。

filter 是选择原 RDD 里所有数据中满足某个特定条件的数据，去返回一个新的 RDD。

mapPartitions 是 map 的变种。不同于 map 的输入函数是应用于 RDD 中每个元素，mapPartitions 的输入函数是应用于 RDD 的每个分区，也就是把每个分区中的内容作为整体来处理的，所以输入函数的类型是 Iterator[T] => Iterator[U]。

groupByKey 和 SQL 中的 groupBy 类似，是把对象的集合按某个 Key 来归类，返回的 RDD 中每个 Key 对应一个序列。

## RDD 的动作操作

RDD 中的动作操作 collect 与函数式编程中的 collect 类似，它会以数组的形式，返回 RDD 的所有元素。需要注意的是，collect 操作只有在输出数组所含的数据数量较小时使用，因为所有的数据都会载入到程序的内存中，如果数据量较大，会占用大量 JVM 内存，导致内存溢出。

与 MapReduce 中的 reduce 类似，它会把 RDD 中的元素根据一个输入函数聚合起来。

Count 会返回 RDD 中元素的个数。仅适用于 Key-Value pair 类型的 RDD，返回具有每个 key 的计数的 的 map。

Spark 在每次转换操作的时候，使用了新产生的 RDD 来记录计算逻辑，这样就把作用在 RDD 上的所有计算逻辑串起来，形成了一个链条。

当对 RDD 进行动作时，Spark 会从计算链的最后一个 RDD 开始，依次从上一个 RDD 获取数据并执行计算逻辑，最后输出结果。

每当我们对 RDD 调用一个新的 action 操作时，整个 RDD 都会从头开始运算。因此，我们应该对多次使用的 RDD 进行一个持久化操作。

## 小结
Spark 在每次转换操作的时候使用了新产生的 RDD 来记录计算逻辑，这样就把作用在 RDD 上的所有计算逻辑串起来形成了一个链条，但是并不会真的去计算结果。

当对 RDD 进行动作 Action 时，Spark 会从计算链的最后一个 RDD 开始，利用迭代函数（Iterator）和计算函数（Compute），依次从上一个 RDD 获取数据并执行计算逻辑，最后输出结果。

此外，我们可以将复杂计算和经常调用的 RDD 进行持久化处理，从而提升计算效率。

## 思考题

对 RDD 进行持久化操作和记录 Checkpoint，有什么区别呢？

主要区别应该是对依赖链的处理：

checkpoint在action之后执行，相当于事务完成后备份结果。既然结果有了，之前的计算过程，也就是RDD的依赖链，也就不需要了，所以不必保存。

但是cache和persist只是保存当前RDD，并不要求是在action之后调用。相当于事务的计算过程，还没有结果。既然没有结果，当需要恢复、重新计算时就要重放计算过程，自然之前的依赖链不能放弃，也需要保存下来。需要恢复时就要从最初的或最近的checkpoint开始重新计算。

区别在于Checkpoint会清空该RDD的依赖关系，并新建一个CheckpointRDD依赖关系，让该RDD依赖，并保存在磁盘或HDFS文件系统中，当数据恢复时，可通过CheckpointRDD读取RDD进行数据计算；持久化RDD会保存依赖关系和计算结果至内存中，可用于后续计算。

从目的上来说，checkpoint用于数据恢复，RDD持久化用于RDD的多次计算操作的性能优化，避免重复计算。从存储位置上看checkpoint储存在外存中，RDD可以根据存储级别存储在内存或/和外存中。