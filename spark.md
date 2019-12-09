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

## 离散流（Discretized Streams(DStream))
`Discretized Streams` 或者 `DStreams`是Spark流提供的基本抽象。它表示连续的数据流，可以是从数据源接收到的输入数据流，也可以是通过转换输入数据流生成的数据流。在内部，DStream由一系列连续的RDDs表示，这是Spark对不可变的分布式数据集的抽象（参见[Spark编程指南](http://spark.apache.org/docs/latest/rdd-programming-guide.html#resilient-distributed-datasets-rdds)了解更多细节。DStream中的每个RDD包含来自某个时间间隔的数据，如下图所示。
![Spark Streaming Data Flow](./assets/streaming-dstream.png)

应用于DStream上的任何操作都转换为底层RDDs上的操作。例如，在前面将一个行流转换为单词的示例中，flatMap操作应用于行DStream中的每个RDD，以生成 words DStream的RDDs。如下图所示。
![Spark Streaming Data Flow](./assets/streaming-dstream-ops.png)

这些底层的RDD转换是由Spark引擎计算的。DStream操作隐藏了这些细节中的大部分，并为开发人员提供了更高级的API。这些操作将在后面的小节中详细讨论。

## 输入数据流和接收器(Input DStreams and Receivers)
输入数据流是表示从数据源接收的输入数据流。在[quick示例](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)中，lines 是一个输入DStream，它表示从netcat服务器接收到的数据流。每个输入DStream(本节后面讨论的文件流除外)都与接收方([Scala doc](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.receiver.Receiver)、[Java doc](http://spark.apache.org/docs/latest/api/java/org/apache/spark/streaming/receiver/Receiver.html))对象相关联，接收方接收来自源的数据并将其存储在Spark内存中进行处理。
Spark Streaming 提供了两类内置流数据源。
 * 基本资源:StreamingContext API中直接可用的资源。示例:文件系统和套接字连接。
 * 高级资源:像Kafka, Flume, Kinesis等资源可以通过额外的工具类获得。如链接部分所述，这些需要针对额外依赖项进行链接。
 我们将在本节后面讨论每个类别中的一些资源。

 注意，如果希望在流应用程序中并行接收多个数据流，可以创建多个输入DStreams(将在性能调优部分进一步讨论)。这将创建多个接收器，同时接收多个数据流。但是请注意，Spark worker/executor是一个长时间运行的任务，因此它占用分配给Spark流应用程序的一个内核。因此，重要的需要记住的是，Spark流应用程序需要分配足够的核心(或线程，如果在本地运行)来处理接收到的数据，以及运行接收方。
**重要点**
 * 在本地运行Spark流程序时，不要使用“local”或“local[1]”作为主URL。这两种方法都意味着只有一个线程将用于在本地运行任务。如果您使用基于接收器的输入DStream(例如sockets、Kafka、Flume等)，那么将使用单个线程来运行接收器，不留下任何线程来处理接收到的数据。因此，在本地运行时，始终使用“local[n]”作为主URL，其中要运行小于n个接收方(有关如何设置主接收方的信息，请参阅[Spark Properties](http://spark.apache.org/docs/latest/configuration.html#spark-properties))。
 * 将逻辑扩展到在集群上运行，分配给Spark Streaming应用程序的内核数量必须大于接收器的数量。否则，系统将接收数据，但但却无法处理它。

 ### 基础源(Basic Sources)

 我们已经在这个[快速的示例中](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)看到了ssc.socketTextStream(…)，它根据通过TCP套接字连接接收的文本数据创建DStream。除了套接字之外，StreamingContext API还提供了将文件创建为输入源的方法。
 #### 文件源(File Streams)
对于从任何与HDFS API(即HDFS、S3、NFS等)兼容的文件系统上的文件中读取数据，可以通过 StreamingContext.fileStream[KeyClass ValueClass, InputFormatClass]创建DStream。

文件流不需要运行接收器，因此不需要为接收文件数据分配任何内核。

对于简单的文本文件，最简单的方法是StreamingContext.textFileStream(dataDirectory)。
```Scala
streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
```
对于文件类型
```Scala
streamingContext.textFileStream(dataDirectory)
```
**如何监控目录**

Spark Streaming将监视目录dataDirectory并处理在该目录中创建的任何文件。
 * 可以监视一个简单的目录，比如“hdfs://namenode:8040/logs/”。位于该路径下的所有文件发现时将被处理。
 * 可以提供[POSIX glob模式](http://pubs.opengroup.org/onlinepubs/009695399/utilities/xcu_chap02.html#tag_02_13_02)，例如“hdfs://namenode:8040/logs/2017/*”。在这里，DStream将包含与模式匹配的目录中的所有文件。也就是说:它是目录的模式，而不是目录中的文件。
 * 所有文件必须采用相同的数据格式。
 * 一个文件被认为是基于其修改时间而不是创建时间的时间段的一部分。
 * 处理后，对当前窗口内文件的更改将不会导致文件被重新读取。也就是说:更新被忽略。
 * 目录下的文件越多，扫描更改所需的时间就越长——即使没有修改任何文件。
 * 如果使用通配符来标识目录，如“hdfs://namenode:8040/logs/2016-*”，则重命名整个目录以匹配路径将该目录添加到受监视的目录列表中。只有修改时间在当前窗口内的目录中的文件才会包含在流中。
 * 调用FileSystem.setTimes()来修正时间戳是在以后的处理窗口中获取文件的一种方法，即使它的内容没有改变。
**使用对象存储作为数据源**
像HDFS这样的“Full”文件系统倾向于在创建输出流时立即设置文件的修改时间。当一个文件被打开时，甚至在数据被完全写入之前，它就被包含在DStream中——在此之后，同一窗口内的文件更新将被忽略。也就是说:变更可能被遗漏，数据可能从流中被遗漏。

要确保在窗口中进行更改，请将文件写入未监视的目录，然后在关闭输出流之后立即将其重命名为目标目录。如果重新命名的文件在处理窗口时间内出现在扫描的目标目录中，则新数据将被处理。

相反，像Amazon S3和Azure Storage这样的对象存储通常有较慢的重命名操作，因为数据实际上是被复制的。此外，重命名的对象可能将rename()操作的时间作为其修改时间，因此可能不被认为是窗口的一部分，而新对象的创建时间意味着它们是窗口的一部分。

需要对目标对象存储进行仔细的测试，以验证存储的时间戳行为与Spark Streaming所期望的一致。直接写入目标目录可能是通过所选对象存储流数据的适当策略。

关于这个主题的更多细节，请参考[Hadoop文件系统规范](https://hadoop.apache.org/docs/stable2/hadoop-project-dist/hadoop-common/filesystem/introduction.html)。

#### 基于自定义接收器的流(Streams based on Custom Receivers)
可以使用通过自定义接收器接收的数据流创建DStreams。有关详细信息，请参阅[自定义接收方](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)指南。

**将RDDs队列作为流**
要使用测试数据测试Spark流应用程序，还可以使用streamingContext.queueStream(queueOfRDDs)创建基于RDDs队列的DStream。推入队列的每个RDD将被视为DStream中的一批数据，并像流一样进行处理。

有关套接字流和文件流的更多信息，请参见Scala的[StreamingContext](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.StreamingContext)、Java的[JavaStreamingContext](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html)和Python的[StreamingContext](http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext)中相关函数的API文档。

### 先进的来源(Advanced Sources)
**Python API** 从Spark 2.4.4开始，在这些源代码中，Kafka、Kinesis和Flume都可以在Python API中找到。

这类源需要与外部非spark库交互，其中一些具有复杂的依赖关系(例如Kafka和Flume)。因此，为了最小化与版本依赖冲突相关的问题，从这些源创建DStreams的功能已经转移到单独的库中，在必要时可以显式地[链接](http://spark.apache.org/docs/latest/streaming-programming-guide.html#linking)到这些库。

注意，这些高级数据源在Spark shell中不可用，因此基于这些高级数据源的应用程序不能在shell中测试。如果您真的想在Spark shell中使用它们，那么您必须下载相应的Maven工件及其依赖项，并将其添加到类路径中。

这些先进的来源如下。

 * **kafka**: Spark Streaming 2.4.4与Kafka代版本0.8.2.1或更高兼容。有关更多细节，请参阅[Kafka集成指南](http://spark.apache.org/docs/latest/streaming-kafka-integration.html)。
 * **Flume**: Spark Streaming 2.4.4与Flume 1.6.0兼容。有关更多细节，请参阅[Flume集成指南](http://spark.apache.org/docs/latest/streaming-flume-integration.html)。
 * **kinesis **: Spark Streaming2.4.4与Kinesis客户端库1.2.1兼容。有关更多细节，请参阅[kinesis集成指南](http://spark.apache.org/docs/latest/streaming-kinesis-integration.html)。
### 定制源(Custom Sources)
**Python API** Python中还不支持这一点。
还可以从自定义数据源创建输入DStreams。您所要做的就是实现一个用户定义的接收器(请参阅下一节了解它是什么)，它可以接收来自自定义源的数据并将其推入Spark。有关详细信息，请参阅自定义接收方指南。

### 接收器的可靠性(Receiver Reliability)

基于数据源的可靠性，可以划分两种数据源。数据源(如Kafka和Flume)允许确认传输的数据。如果从这些可靠来源接收数据的系统正确地确认接收到的数据，则可以确保不会由于任何类型的故障而丢失任何数据。这就导致了两种类型的接受者:
 1. *可靠的接收器* —— 一个可靠的接收器发送确认到一个可靠的来源时，数据已被分片接收和存储在Spark。
 2. *不可靠的接收方* —— 不可靠的接收方不会向源发送确认信息。这用于不支持确认的源，甚至可以用于不希望或不需要进入确认复杂性的可靠源。

如何编写可靠的接收器的详细信息在[自定义接收器](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)指南中进行了讨论。

## 转换DStreams(Transformations on DStreams)
与RDDs类似，转换允许修改输入DStream中的数据。DStreams支持许多在普通Spark RDD上可用的转换。一些常见的操作如下。
转换|意义
-|-
map(func)|通过函数func转换源DStream的每个元素来返回一个新的DStream。
flatMap(func)|与map类似，但是每个输入项可以映射到0或多个输出项。
filter(func)|通过只选择func返回true的源DStream的记录来返回一个新的DStream。
repartition(numPartitions)|通过创建更多或更少的分区来改变DStream中的并行度。
union(otherStream)|返回一个新的DStream，它包含源DStream和otherDStream中元素的并集。
