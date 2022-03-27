# 15 | Spark SQL：Spark数据查询的利器

Spark SQL 不仅将关系型数据库的处理模式和 Spark 的函数式编程相结合，还兼容多种数据格式，包括 Hive、RDD、JSON 文件、CSV 文件等。

## Spark SQL 的架构

Spark SQL 本质上是一个库。它运行在 Spark 的核心执行引擎之上。

![img](https://static001.geekbang.org/resource/image/3b/13/3bdb29b1d697e3530d1efbd05e694e13.png)

如上图所示，它提供类似于 SQL 的操作接口，允许数据仓库应用程序直接获取数据，允许使用者通过命令行操作来交互地查询数据，还提供两个 API：DataFrame API 和 DataSet API。

与基本的 Spark RDD API 不同，Spark SQL 提供的接口为 Spark 提供了关于数据结构和正在执行的计算的更多信息。

## DataSet

DataSet 上的转换操作也不会被立刻执行，只是先生成新的 DataSet，只有当遇到动作操作，才会把之前的转换操作一并执行，生成结果。

所以，DataSet 的内部结构包含了逻辑计划，即生成该数据集所需要的运算。当动作操作执行时，Spark SQL 的查询优化器会优化这个逻辑计划，并生成一个可以分布式执行的、包含分区信息的物理计划。

![img](https://static001.geekbang.org/resource/image/5a/e3/5a6fd6e91c92c166d279711bf9c761e3.png)

如上图所示，左侧的 RDD 虽然以 People 为类型参数，但 Spark 框架本身不了解 People 类的内部结构。所有的操作都以 People 为单位执行。

而右侧的 DataSet 却提供了详细的结构信息与每列的数据类型。

这让 Spark SQL 可以清楚地知道该数据集中包含哪些列，每列的名称和类型各是什么。也就是说，DataSet 提供数据表的 schema 信息。这样的结构使得 DataSet API 的执行效率更高。

其次，由于 DataSet 存储了每列的数据类型。所以，在程序编译时可以执行类型检测。

## DataFrame

DataFrame 可以被看作是一种特殊的 DataSet。它也是关系型数据库中表一样的结构化存储机制，也是分布式不可变的数据结构。

但是，它的每一列并不存储类型信息，所以在编译时并不能发现类型错误。DataFrame 每一行的类型固定为 Row，他可以被当作 DataSet[Row]来处理，我们必须要通过解析才能获取各列的值。

## RDD、DataFrame、DataSet 对比

![img](https://static001.geekbang.org/resource/image/40/ef/40691757146e1b480e08969e676644ef.png)

不变性与分区由于 DataSet 和 DataFrame 都是基于 RDD 的，所以它们都拥有 RDD 的基本特性，在此不做赘述。而且我们可以通过简单的 API 在 DataFrame 或 Dataset 与 RDD 之间进行无缝切换。

Spark 程序运行时，Spark SQL 中的查询优化器会对语句进行分析，并生成优化过的 RDD 在底层执行。

错误检测RDD 和 DataSet 都是类型安全的，而 DataFrame 并不是类型安全的。这是因为它不存储每一列的信息如名字和类型。

## 小结

DataFrame 和 DataSet 是 Spark SQL 提供的基于 RDD 的结构化数据抽象。
它既有 RDD 不可变、分区、存储依赖关系等特性，又拥有类似于关系型数据库的结构化信息。
所以，基于 DataFrame 和 DataSet API 开发出的程序会被自动优化，使得开发人员不需要操作底层的 RDD API 来进行手动优化，大大提升开发效率。
但是 RDD API 对于非结构化的数据处理有独特的优势，比如文本流数据，而且更方便我们做底层的操作。所以在开发中，我们还是需要根据实际情况来选择使用哪种 API。

## 思考题

什么场景适合使用 DataFrame API，什么场景适合使用 DataSet API？

待处理数据的schema是静态的且对其完全掌控的情况下用DataSet，反之用DataFrame