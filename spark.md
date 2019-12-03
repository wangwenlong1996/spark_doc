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
将 `words` DStream进一步`map`（一对一转换）到（word，1）对的DStream中，然后将其`reduce`以获取每批数据中单词的频率。最后，wordCounts.print（）将打印每秒生成的一些计数。

请注意，执行这些行时，Spark Streaming仅设置启动时将执行的计算，但是尚未开始任何实际处理。我们最终可以执行
```
ssc.start()             // Start the computation
ssc.awaitTermination()  // Wait for the computation to terminate
```
完整的代码可以在Spark Streaming示例[NetworkWordCount](https://github.com/apache/spark/blob/v2.4.4/examples/src/main/scala/org/apache/spark/examples/streaming/NetworkWordCount.scala)中找到。

如果您已经下载并构建了Spark，则可以按以下方式运行此示例。您首先需要通过使用以下命令将Netcat（在大多数类Unix系统中找到的一个小实用程序）作为数据服务器运行
```
$ nc -lk 9999
```
然后，在另一个终端中，您可以通过使用
```
./bin/run-example streaming.NetworkWordCount localhost 9999
```
然后，将对运行netcat服务器的终端中键入的任何行进行计数并每秒打印一次。它将类似于以下内容。
```
# TERMINAL 1:
# Running Netcat

$ nc -lk 9999

hello world



...
```
``` scala
# TERMINAL 2: RUNNING NetworkWordCount

$ ./bin/run-example streaming.NetworkWordCount localhost 9999
...
-------------------------------------------
Time: 1357008430000 ms
-------------------------------------------
(hello,1)
(world,1)
...
```
# 基本概念
接下来，我们将脱离简单的示例，并详细介绍Spark Streaming的基础知识。
## 链接（Linking)
与Spark相似，可以通过Maven Central使用Spark Streaming。要编写自己的Spark Streaming程序，您必须将以下依赖项添加到SBT或Maven项目中。
```maven
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.12</artifactId>
    <version>2.4.4</version>
    <scope>provided</scope>
</dependency>
```
```sbt
libraryDependencies += "org.apache.spark" % "spark-streaming_2.12" % "2.4.4" % "provided"
```

要从Kafka、Flume和Kinesis等Spark Streaming核心API中没有的数据源中获取数据，您必须将相应的包 spark-streaming-xyz_2.12添加到依赖项中。例如，一些常见的例子如下。
Source | Artifact
-| -
Kafka|spark-streaming-kafka-0-10_2.12
Flume|spark-streaming-flume_2.12
Kinesis|spark-streaming-kinesis-asl_2.12 [Amazon Software License]
要获得最新的列表，请参考Maven存储库，以获得受支持的源代码和包的完整列表。

## 初始化StreamingContext
要初始化一个Spark Streaming程序，必须创建一个`StreamingContext`对象，该对象是所有Spark流功能的主要入口点。

### scala
可以从SparkConf对象创建StreamingContext对象。
```scala
import org.apache.spark._
import org.apache.spark.streaming._

val conf = new SparkConf().setAppName(appName).setMaster(master)
val ssc = new StreamingContext(conf, Seconds(1))
```

`appName`参数是应用程序在集群UI上显示的名称。master是一个Spark、Mesos、Kubernetes或YARN集群的URL，或者一个特殊的`local[*]`字符串，在本地模式下运行。实际上，在集群上运行时，您不希望在程序中硬编码master，而是使用spark-submit启动应用程序并在那里接收它。但是，对于本地测试和单元测试，您可以通过`local[*]`来运行Spark Streaming 进程(检测本地系统中的内核数量)。注意，这在内部创建了一个SparkContext(所有Spark功能的起点)，它可以作为ssc.sparkContext访问。

批处理间隔必须根据应用程序的延迟需求和可用的集群资源来设置。有关更多细节，请参阅性能调优部分。

Streaming Context对象也可以从现有的SparkContext对象创建。
```scala
import org.apache.spark.streaming._

val sc = ...                // existing SparkContext
val ssc = new StreamingContext(sc, Seconds(1))
```
 1. 定义上下文之后，您必须执行以下操作。
 2. 通过创建输入DStreams来定义输入源。
 3. 通过对DStreams应用转换和输出操作来定义流计算。
 4. 开始接收数据并使用streamingContext.start()进行处理。
 5. 使用streamingContext.awaitTermination()等待处理停止(手动或由于任何错误)。
 6. 可以使用streamingContext.stop()手动停止处理。

注意点：
 * 一旦上下文（context）启动，就不能设置或添加新的流计算。
 * 上下文(context)一旦停止，就不能重新启动。
 * JVM中只能同时激活一个StreamingContext。
 * StreamingContext上的stop()也会停止SparkContext。要仅停止StreamingContext，请将名为stopSparkContext的stop()的可选参数设置为false。
 * 只要在创建下一个StreamingContext之前停止前一个StreamingContext(不停止SparkContext)，就可以重用SparkContext来创建多个StreamingContext。

## 离散流（DStream)
