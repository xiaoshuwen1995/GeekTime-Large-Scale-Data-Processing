# 39 | 从SQL到Streaming SQL：突破静态数据查询的次元

流处理之所以重要，是因为现在是个数据爆炸的时代，大部分数据源是每时每刻都在更新的。数据处理系统对时效性的要求都很高。

当说到批处理的时候，我们第一个想到的工具就是 SQL，因为SQL语法简单易懂，方便使用。

Spark 的 Structured Streaming 就是用支持类 SQL 的 DataFrame API 去做流处理的，Flink、Storm也支持用类似于 SQL 处理流数据。但它们也都是把 SQL 做成 API 和其他的处理逻辑结合在一起，并没有把它单独分离成一种语言、为它定义语法。

有没有类似 SQL 语法来对流数据进行查询的语言呢？答案是肯定的。我们把这种语言统称为 Streaming SQL。

Siddhi Streaming SQL 和 Kafka KSQL 就是两个典型的 Streaming SQL 语言。

不同于 SQL，Streaming SQL 并没有统一的语法规范，不同平台提供的 Streaming SQL 语法都有所不同。

 Streaming SQL 作用的数据对象也不是有界的数据表，而是无边界的数据流。无边界的数据流可以被认为是底部不断新增数据的一张表。

Streaming SQL 允许我们用类似于 SQL 的命令形式去处理无边界的流数据，它有如下几个优点：

简单易学，使用方便：SQL 可以说是流传最广泛的数据处理语言，对大部分人来说，Streaming SQL 的学习成本很低。

效率高，速度快：SQL 问世这么久，它的执行引擎已经被优化过很多次，很多 SQL 的优化准则被直接借鉴到 Streaming SQL 的执行引擎上。代码简洁，而且涵盖了大部分常用的数据操作。

## 数据映射、过滤、聚合

SQL 是一个强大的、对有结构数据进行查询的语言，它提供几个独立的操作，如数据映射（SELECT）、数据过滤（WHERE）、数据聚合（GROUP BY）和数据联结（JOIN）。将这些基本操作组合起来，可以实现很多复杂的查询。

在 Streaming SQL 中， SELECT 和 WHERE 是必备功能。数据映射就是从流中取出数据的一部分属性，作为新的流输出，它定义了输出流的格式。数据过滤就是根据某些属性，挑选出符合条件的数据。

```sql
Select id, t*7/5 + 32 as tF from BoilerStream[t > 350];  //Siddhi Streaming SQL

Select id, t*7/5 + 32 as tF from BoilerStream Where t > 350; //Kafka KSQL
```

除了上面提到过的数据映射和数据过滤，Streaming SQL 的 GROUP BY 也和 SQL 中的用法类似。接下来，让我们一起了解 Streaming SQL 的其他重要操作：窗口（Window）、联结（Join）和模式（Pattern）。

## 窗口

在之前 Spark 和 Beam 的流处理章节中，我们都学习过窗口的概念。窗口，就是把流中的数据按照时间戳划分成一个个有限的集合。在此之上，我们可以统计各种聚合属性如平均值等。

在现实世界中，大多数场景下我们只需要关心特定窗口，而不需要研究全局窗口内的所有数据，这是由数据的时效性决定的。应用最广的窗口类型是以当前时间为结束的滑动窗口

```sql
Select bid, avg(t) as T From BoilerStream#window.length(10) insert into BoilerStreamMovingAveage; // Siddhi Streaming SQL

Select bid, avg(t) as T From BoilerStream WINDOW HOPPING (SIZE 10, ADVANCE BY 1); // Kafka KSQL
```

在 Beam Window 中，我们介绍过固定窗口和滑动窗口的区别，而每种窗口都可以是基于时间或数量的，所以就有 4 种组合：

- 滑动时间窗口：统计最近时间段 T 内的所有数据，每当经过某个时间段都会被触发一次。
- 固定时间窗口：统计最近时间段 T 内的所有数据，每当经过 T 都会被触发一次。
- 滑动长度窗口：统计最近 N 个数据，每当接收到一个（或多个）数据都会被触发一次。
- 固定长度窗口：统计最近 N 个数据，每当接收到 N 个数据都会被触发一次。

## 联结

当我们要把两个不同流中的数据通过某个属性连接起来时，就要用到 Join。由于在任一个时刻，流数据都不是完整的，第一个流中后面还没到的数据有可能要和第二个流中已经有的数据 Join 起来再输出。所以，对流的 Join 一般要对至少一个流附加窗口，这也和第 20 讲中提到的数据水印类似。

```sql
from TempStream[temp > 30.0]#window.time(1 min) as T
  join RegulatorStream[isOn == false]#window.length(1) as R
  on T.roomNo == R.roomNo
select T.roomNo, R.deviceID, 'start' as action
insert into RegulatorActionStream; // Siddhi Streaming SQL
```

想要得到所有温度高于 30 度但是空调没打开的的房间，从而把它们的空调打开降温

## 模式

通过上面的介绍我们可以看出，Streaming SQL 的数据模型继承自 SQL 的关系数据模型，唯一的不同就是每个数据都有一个时间戳，并且每个数据都是假设在这个特定的时间戳才被接收。那么我们很自然地就会想研究这些数据的顺序，比如，事件 A 是不是发生在事件 B 之后？这类先后顺序的问题在日常生活中很普遍。这里你不难看出，我们其实是在检测某个模式有没有在特定的时间段内发生。

在流数据处理中，检测模式是一类重要的问题，我们会经常需要通过对最近数据的研究去总结发展的趋势，从而“预测”未来。

```sql
from every( e1=TempStream ) -> e2=TempStream[ e1.roomNo == roomNo and (e1.temp + 5) <= temp ]
    within 10 min
select e1.roomNo, e1.temp as initialTemp, e2.temp as finalTemp
insert into AlertStream;
```

在 Siddhi Streaming SQL 中，“->”这个操作符用于声明发生的先后关系。这个 query 检测的模式是 10 分钟内房间温度上升超过 5 度，就插入到输出流。

## 小结

今天我们初步了解了 Streaming SQL 语言的基本功能。虽然没有统一的语法规范，但是各个 Streaming SQL 语言都支持继承自SQL的相似操作符，如数据映射、数据过滤、联结、窗口和模式等。Streaming 中的模式是独有的，这是由于流数据天生具有的时间性所导致。

Streaming SQL 大大降低了开发人员实现流处理的难度，让流处理变得就像写 SQL 查询语句一样简单。它现在还在高速发展，相信未来会变得越来越普遍。