# 总览
Spark Streaming 是Spark core API的扩展，支持可伸缩、高吞吐量、容错的实时数据流处理。数据可以从许多来源获取，如Kafka、Flume、Kinesis或TCP sockets，可以使用复杂的算法处理数据，这些算法用高级函数表示，如map、reduce、join和window。最后，处理后的数据可以推送到文件系统、数据库和活动仪表板。实际上，您可以将Spark的机器学习和图处理算法应用于数据流。
@import "assets/spark_streaming_1.png"{width="300px" height="200px" title="Spark Streaming architecture" alt=""}
在内部，它是这样工作的。Spark Streaming接受实时输入数据流，并将数据分成批次，然后由Spark engine处理，以批量生成最终的结果流。
@import "assets/spark_streaming-flow.png"{width="300px" height="200px" title="Spark Streaming data flow" alt=""}
Spark Streaming提供了一种高级抽象，称为discretized stream或DStream，用来表示连续的数据流。DStreams可以从Kafka、Flume和Kinesis等源的输入数据流创建，也可以通过对其他DStreams应用高级操作创建。在内部，DStream表示为RDDs序列。

本指南向您展示了如何使用DStreams编写Spark流程序。您可以用Scala、Java或Python(在Spark 1.2中引入)编写Spark Streaming程序，所有这些都在本指南中介绍。在本指南中，您可以找到选项卡，让您在不同语言的代码片段之间进行选择。

**Note**:有一些api是不同的，或者在Python中不可用的。在本指南中，您将发现标记Python API突出了这些差异。

# 快速入门例子
在详细介绍如何编写自己的Spark Streaming程序之前，让我们先快速了解一下简单的Spark Streaming程序是什么样子的。假设我们要计算从监听TCP套接字的数据服务器接收到的文本数据中的字数。你所需要做的就是如下所示。
## scala
首先，我们将Spark流类的名称和一些从StreamingContext的隐式转换导入到我们的环境中，以便向我们需要的其他类(如DStream)添加有用的方法。StreamingContext是所有流功能的主要入口点。我们用两个执行线程创建一个本地StreamingContext，批处理间隔为1秒。

```scala
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3

// Create a local StreamingContext with two working thread and batch interval of 1 second.
// The master requires 2 cores to prevent a starvation scenario.

val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
```
使用这个上下文(context)，我们可以创建一个表示来自TCP源的流数据DStream，指定为主机名(例如localhost)和端口(例如9999)
```scala
// Create a DStream that will connect to hostname:port, like localhost:9999
val lines = ssc.socketTextStream("localhost", 9999)
```
这 lines DStream表示从数据服务器接收的数据流。DStream中的每个记录都是一行文本。接下来，我们希望按空格字符将行分割为单词。
```scala
// Split each line into words
val words = lines.flatMap(_.split(" "))
```
flatMap是一个一对多的DStream操作，它通过从源DStream中的每个记录生成多个新记录来创建一个新的DStream。在这种情况下，每一行将被分成多个单词，单词流表示为单词DStream。接下来，我们要计数这些单词。
```scala
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3
// Count each word in each batch
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)

// Print the first ten elements of each RDD generated in this DStream to the console
wordCounts.print()
```
