# 18 | Word Count：从零开始运行你的第一个Spark应用

由于之前代码中创建的 df_lines 这个 DataFrame 中，每一行只有一列，每一列都是一个包含很多词语的句子，我们可以先对这一列做 split，生成一个新的列，列中每个元素是一个词语的数组；再对这个列做 explode，可以把数组中的每个元素都生成一个新的 Row。这样，就实现了类似的 flatMap 功能。这个过程可以用下面的三个表格说明。接下来我们只需要对 Word 这一列做 groupBy，就可以统计出每个词语出现的频率，代码如下。

![img](https://static001.geekbang.org/resource/image/c9/55/c9ebf1f324a73539a2a57ce8151e4455.png)

```Python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

if __name__ == "__main__":
   spark = SparkSession
       .builder
       .appName(‘WordCount’)
       .getOrCreate()
   lines = spark.read.text("sample.txt")
   wordCounts = lines
       .select(explode(split(lines.value, " "))
       .alias("word"))
       .groupBy("word")
       .count()
   wordCounts.show()

   spark.stop()
```

从这个例子，你可以很容易看出使用 DataSet/DataFrame API 的便利性——我们不需要创建（word, count）的 pair 来作为中间值，可以直接对数据做类似 SQL 的查询。