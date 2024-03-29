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
count()|通过计算源DStream的每个RDD中的元素数量，返回一个新的单元素RDDs DStream。
reduce(func)|通过使用函数func(接受两个参数并返回一个参数)聚合源DStream的每个RDD中的元素，返回一个新的单元素RDDs DStream。这个函数应该是结合律和交换律，这样才能并行计算。
countByValue()|当对类型为K的元素的DStream调用时，返回一个新的DStream (K, Long)对，其中每个键的值是它在源DStream的每个RDD中的频率。
reduceByKey(func, [numTasks])|当在(K, V)对的DStream上调用时，返回一个新的(K, V)对的DStream，其中每个键的值使用给定的reduce函数进行聚合。注意:在默认情况下，这将使用Spark的默认并行任务数(本地模式为2，而在集群模式下，该数量由配置属性Spark .default.parallelism决定)来进行分组。您可以传递一个可选的numTasks参数来设置不同数量的任务。
join(otherStream, [numTasks])|当调用两个DStream (K, V)和(K, W)对时，返回一个新的DStream (K， (V, W))对，包含每个键的所有对的元素。
cogroup(otherStream, [numTasks])|当调用(K, V)和(K, W)对的DStream时，返回一个新的(K, Seq[V]， Seq[W])元组DStream。
transform(func)|通过对源DStream的每个RDD应用一个RDD-to-RDD函数来返回一个新的DStream。这可以用来在DStream上执行任意的RDD操作。
updateStateByKey(func)|返回一个新的“state”DStream，其中通过对键的前一个状态和键的新值应用给定的函数来更新每个键的状态。这可以用来维护每个键的状态数据。
其中一些转换值得更详细地讨论。
### UpdateStateByKey操作
`updateStateByKey`操作允许您维护任意状态，同时不断地用新信息更新它。要使用它，您必须执行两个步骤。
 1. 定义状态——状态可以是任意的数据类型。
 2. 定义状态更新函数——使用一个函数指定如何使用输入流中的前一个状态和新值来更新状态。

在每个批处理中，Spark将对所有现有键应用状态更新功能，而不管它们在批处理中是否有新数据。如果更新函数返回None，则键值对将被删除。

让我们用一个例子来说明这一点。假设您希望维护在文本数据流中看到的每个单词的运行计数。这里，运行计数是状态，它是一个整数。我们将更新函数定义为:
```Scala
def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount = ...  // add the new values with the previous running count to get the new count
    Some(newCount)
}
```
这将应用于包含单词的DStream(例如，[前面示例](https://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)中包含(word，1)对的DStream)。
```Scala
val runningCounts = pairs.updateStateByKey[Int](updateFunction _)
```
每个单词都将调用update函数，newValues的序列为1(来自(word, 1)对)，runningCount的序列为前一个计数。
注意，使用updateStateByKey需要配置检查点目录，检查点一节将对此进行详细讨论。

### 转换运算(Transform Operation)
`transform`操作(及其变体，如`transformWith`)允许在DStream上应用任意的RDD-to-RDD函数。它可以用于应用DStream API中没有公开的任何RDD操作。例如，将数据流中的每个批处理与另一个`dataset` join的功能并没有直接在DStream API中公开。但是，您可以很容易地使用transform来实现这一点。这带来了非常强大的可能性。例如，可以通过将输入数据流与预先计算的垃圾邮件信息(也可以使用Spark生成)连接起来，然后根据这些信息进行过滤，从而进行实时数据清理。
```Scala
val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD containing spam information

val cleanedDStream = wordCounts.transform { rdd =>
  rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to do data cleaning
  ...
}
```
注意，提供的函数在每个批处理间隔中被调用。这允许您执行时变换操作RDD，即RDD操作、分区数量、广播变量等可以在批处理期间更改。

### 窗口操作(Window Operations)
Spark Streaming还提供了窗口计算，它允许您在数据的滑动窗口上应用转换。下图演示了这个滑动窗口。
![streaming-dstream-window](assets/streaming-dstream-window.png)

如图所示，每当窗口在源DStream上滑动时，位于窗口内的源RDDs就会被执行合并操作，以生成窗口化的DStream的RDDs。在本例中，操作应用于数据的3个时间单位，滑动2个时间单位。这表明任何窗口操作都需要指定两个参数。
 * 窗口长度 -— 窗口的持续时间(图中为3)。
 * 滑动时间间隔 -- 窗口操作执行的时间间隔(图中为2)。
这两个参数必须是源DStream的批处理间隔的倍数(图中为1)。

让我们用一个例子来说明窗口操作。例如，您希望通过每10秒在最后30秒的数据中生成字数计数来扩展[前面的示例](https://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)。为此，我们必须在最后30秒的数据中对(word, 1)对的DStream应用reduceByKey操作。这是使用reduceByKeyAndWindow操作完成的。
```scala
// Reduce last 30 seconds of data, every 10 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
```
下面是一些常见的窗口操作。所有这些操作都采用上述两个参数——窗口长度和滑动间隔。
操作|意义
-|-
window(windowLength, slideInterval)|返回一个新的DStream，它是基于源DStream的加窗批量计算的。
countByWindow(windowLength, slideInterval)|返回流中元素的滑动窗口计数。
reduceByWindow(func, windowLength, slideInterval)|返回一个新的单元素流，它是通过使用func将流中的元素在一个滑动区间内聚合而创建的。这个函数应该是结合律和交换律，这样才能正确地并行计算。
reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks])|当在一个(K, V)对的DStream上调用时，返回一个新的(K, V)对的DStream，每个键的值使用指定的reduce函数func在滑动窗口中进行聚合。注意:默认情况下，这将使用Spark的默认并行任务数(本地模式为2，而在集群模式下，该数值由配置属性spark.default.parallelism决定)进行分组。您可以传递一个可选的numTasks参数来设置不同数量的任务。
reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks])|上面的reduceByKeyAndWindow()的一个更有效的版本，其中每个窗口的reduce值是使用前一个窗口的reduce值递增计算的。这是通过减少进入滑动窗口的新数据和“反向减少”离开窗口的旧数据来实现的。例如，在窗口滑动时“添加”和“减去”键数。但是，它只适用于“可逆约简函数”，即具有相应“逆约简”函数的约简函数(取参数invFunc)。与reduceByKeyAndWindow一样，reduce任务的数量可以通过一个可选参数进行配置。注意，必须启用检查点才能使用此操作。
countByValueAndWindow(windowLength, slideInterval, [numTasks])|当调用一个(K, V)对的DStream时，返回一个新的(K, Long)对的DStream，其中每个键的值是它在滑动窗口中的频率。与reduceByKeyAndWindow一样，reduce任务的数量可以通过一个可选参数进行配置。

### Join操作(Join Operations)
最后，值得强调一下在Spark Streaming中执行不同类型的`Join`是多么容易。
**Stream-stream连接**
Streams可以很容易地与其他Streams连接。
```Scala
val stream1: DStream[String, String] = ...
val stream2: DStream[String, String] = ...
val joinedStream = stream1.join(stream2)
```
在这里，在每个批处理间隔中，由stream1生成的RDD将与由stream2生成的RDD相连接。你也可以使用leftOuterJoin, rightOuterJoin, fullOuterJoin。此外，在流的窗口上进行连接通常非常有用。这也很简单。
```Scala
val windowedStream1 = stream1.window(Seconds(20))
val windowedStream2 = stream2.window(Minutes(1))
val joinedStream = windowedStream1.join(windowedStream2)
```
**Stream-dataset连接**
在前面的说明DStream.transform 中已经显示了这一点。变换操作。下面是另一个将窗口化的流与数据集连接的示例。
```Scala
val dataset: RDD[String, String] = ...
val windowedStream = stream.window(Seconds(20))...
val joinedStream = windowedStream.transform { rdd => rdd.join(dataset) }
```
实际上，您还可以动态地更改要加入的数据集。提供的转换函数在每个批处理间隔进行评估，将使用数据集引用点指向的当前数据集。

DStream转换的完整列表在API文档中提供。有关Scala API，请参见[DStream](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.DStream)和[PairDStreamFunctions](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions)。有关Java API，请参见[JavaDStream](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html)和[JavaPairDStream](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html)。有关Python API，请参阅[DStream](https://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.DStream)。

## DStreams上的输出操作(Output Operations on DStreams)

输出操作允许将DStream的数据推送到外部系统，如数据库或文件系统。由于输出操作实际上允许外部系统使用转换后的数据，因此它们会触发所有DStream转换的实际执行(类似于RDDs的actions)。目前定义了以下输出操作:
输出操作| 意义
-|-
print()| 在运行流应用程序的驱动节点上打印DStream中每批数据的前十个元素。这对于开发和调试非常有用。`Python API`这在Python API中称为pprint()。
saveAsTextFiles(prefix, [suffix])| 将DStream的内容保存为文本文件。每个批处理间隔的文件名是根据前缀和后缀“prefix-TIME_IN_MS[.suffix]”生成的。
saveAsObjectFiles(prefix, [suffix])|将这个DStream的内容保存为序列化的Java对象的序列文件。每个批处理间隔的文件名是根据前缀和后缀“prefix-TIME_IN_MS[.suffix]”生成的。`Python API` 这在Python API中不可用。
saveAsHadoopFiles(prefix, [suffix])|将DStream的内容保存为Hadoop文件。每个批处理间隔的文件名是根据前缀和后缀“prefix-TIME_IN_MS[.suffix]”生成的。
foreachRDD(func)|将函数func应用于从流生成的每个RDD的最通用的输出操作符。该函数应该将每个RDD中的数据推送到外部系统，例如将RDD保存到文件中，或者通过网络将其写入数据库。请注意，func函数是在运行流应用程序的驱动程序进程中执行的，并且通常会有RDD动作(action)，这将强制流RDDs的计算。

### 使用foreachRDD的设计模式(Design Patterns for using foreachRDD)
输出操作允许将DStream的数据推送到外部系统，如数据库或文件系统。然而，理解如何正确和有效地使用这个原语是很重要的。要避免的一些常见错误如下。

通常，将数据写入外部系统需要创建一个连接对象(例如，到远程服务器的TCP连接)并使用它将数据发送到远程系统。为此，开发人员可能会无意中尝试在Spark驱动程序中创建连接对象，然后尝试在Spark worker中使用它来保存RDDs中的记录。例如(在Scala中)。

```Scala
dstream.foreachRDD { rdd =>
  val connection = createNewConnection()  // executed at the driver
  rdd.foreach { record =>
    connection.send(record) // executed at the worker
  }
}
```

这是不正确的，因为这需要将连接对象序列化并从驱动程序发送到工作程序。这样的连接对象很少能跨机器转移。此错误可能表现为序列化错误(连接对象不可序列化)、初始化错误(连接对象需要在工作人员处初始化)等。正确的解决方案是在worker上创建连接对象。

然而，这可能会导致另一个常见错误——为每个记录创建一个新连接。例如,
```Scala
dstream.foreachRDD { rdd =>
  rdd.foreach { record =>
    val connection = createNewConnection()
    connection.send(record)
    connection.close()
  }
}
```

通常，创建连接对象需要时间和资源开销。因此，为每个记录创建和销毁一个连接对象可能导致不必要的高开销，并可能显著降低系统的总体吞吐量。更好的解决方案是使用rdd.foreachPartition——创建一个连接对象，并使用该连接发送RDD分区中的所有记录。
```Scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    val connection = createNewConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    connection.close()
  }
}
```

这会将创建连接的开销分摊到许多记录上。

最后，可以通过跨多个RDDs/batch重用连接对象来进一步优化。可以维护一个静态的连接对象池，在将多个批的RDDs推送到外部系统时可以重用这些对象，从而进一步减少开销。
```Scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

请注意，池中的连接应该按需延迟创建，如果不使用一段时间就会超时。这实现了向外部系统发送数据的最高效。
其他注意事项:
 * DStreams的输出操作是延迟执行，就像RDD操作延迟执行一样。具体来说，DStream输出操作中的RDD操作强制处理接收到的数据。因此，如果您的应用程序没有任何输出操作，或者有像dstream.foreachRDD()这样的输出操作，但是其中没有任何RDD操作，那么什么也不会执行。系统将简单地接收数据并丢弃它。
  * 默认情况下，一次执行一个输出操作。它们是按照在应用程序中定义的顺序执行的。

## DataFrame和SQL操作(DataFrame and SQL Operations)

您可以轻松地在流数据上使用[DataFrames和SQL](https://spark.apache.org/docs/latest/sql-programming-guide.html)操作。 您必须使用StreamingContext正在使用的SparkContext创建SparkSession。此外，这样做可以在驱动程序失败时重新启动。这是通过创建一个延迟实例化的SparkSession单例来实现的。如下面的例子所示。它修改了前面的字数统计示例，以使用DataFrames和SQL生成字数统计。每个RDD都被转换为一个DataFrame，注册为一个临时表，然后使用SQL进行查询。
```Scala
val words: DStream[String] = ...

words.foreachRDD { rdd =>

  // Get the singleton instance of SparkSession
  val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()
  import spark.implicits._

  // Convert RDD[String] to DataFrame
  val wordsDataFrame = rdd.toDF("word")

  // Create a temporary view
  wordsDataFrame.createOrReplaceTempView("words")

  // Do word count on DataFrame using SQL and print it
  val wordCountsDataFrame =
    spark.sql("select word, count(*) as total from words group by word")
  wordCountsDataFrame.show()
}
```
查看完整的[源代码](https://github.com/apache/spark/blob/v2.4.4/examples/src/main/scala/org/apache/spark/examples/streaming/SqlNetworkWordCount.scala)。

您还可以执行SQL查询在不同线程(即与运行中的StreamingContext异步)的流数据上定义的表。必须确保您设置了StreamingContext来记住足够数量的流数据，以便查询可以运行。则，StreamingContext(它不知道任何异步SQL查询)将在查询完成之前删除旧的流数据。例如，如果您想要查询上一批数据，但是您的查询可能需要5分钟才能运行完成，那么可以调用streamingContext.remember(minutes(5))(在Scala中，或在其他语言中等效)。

参见{DataFrames和SQL](https://spark.apache.org/docs/latest/sql-programming-guide.html)指南以了解更多关于DataFrames的信息。

## MLlib操作(MLlib Operations)

您还可以轻松使用[MLlib](https://spark.apache.org/docs/latest/ml-guide.html)提供的机器学习算法。首先，有流式机器学习算法(如[流式线性回归](https://spark.apache.org/docs/latest/mllib-linear-methods.html#streaming-linear-regression)、[流式KMeans](https://spark.apache.org/docs/latest/mllib-clustering.html#streaming-k-means)等)可以同时学习流式数据，也可以将模型应用到流式数据上。除此之外，对于更大类别的机器学习算法，您可以离线学习一个算法模型(即使用历史数据)，然后在线将该模型应用于流数据。有关更多细节，请参阅[MLlib](https://spark.apache.org/docs/latest/ml-guide.html)指南。

## 缓存/持久性(Caching / Persistence)

与RDDs类似，DStreams还允许开发人员将流的数据持久化到内存中。也就是说，在DStream上使用persist()方法将自动在内存中持久化该DStream的每个RDD。如果DStream中的数据将被多次计算(例如，对同一数据的多次操作)，那么这是非常有用的。对于基于窗口的操作，如reduceByWindow和reduceByKeyAndWindow，以及基于状态的操作，如updateStateByKey，这是隐式正确的。因此，由基于窗口的操作生成的DStreams将自动持久化到内存中，而无需开发人员调用persist()。

对于通过网络接收数据的输入流(例如，Kafka、Flume、sockets等)，默认的持久性级别被设置为将数据复制到两个节点以实现容错。

注意，与RDDs不同，DStreams的默认持久性级别是在内存中序列化数据。这将在[性能调优](https://spark.apache.org/docs/latest/streaming-programming-guide.html#memory-tuning)一节中进一步讨论。有关不同持久性级别的更多信息可以在[Spark编程指南](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)中找到。

## 检查点(checkpointing)

流应用程序必须7*24小时运行，因此必须对与应用程序逻辑无关的故障具有弹性(例如，系统故障、JVM崩溃等)。为了实现这一点，Spark Streaming需要将足够的信息检查点存储到容错存储系统，以便从故障中恢复。默认有两种类型的数据检查点。

 * **元数据的检查点** -- 将定义流计算的信息保存到容错存储器(如HDFS)。这用于从运行流应用程序的驱动程序的节点的故障中恢复(稍后将详细讨论)。元数据包括:
     * 配置——用于创建流应用程序的配置。
     * DStream操作——定义流应用程序的一组DStream操作。
     * 未完成的批——其作业在队列中但尚未完成的批数据。
  * 数据检查点——将生成的RDDs保存到可靠的存储中。在跨多个批组合数据的一些有状态转换中，这是必需的。在这种转换中，生成的RDDs依赖于前一批的RDDs，这导致依赖链的长度随时间不断增加。为了避免这种恢复时间的无界增长(与依赖链成比例)，有状态转换的中间rdd会定期检查可靠存储(如HDFS)，以切断依赖链。

总之，元数据检查点主要用于从驱动程序故障中恢复，而数据或RDD检查点对于使用有状态转换的基本功能也是必要的。

### 何时启用检查点(When to enable Checkpointing)

有下列任何一项要求的应用必须启用检查点:
 * 使用有状态转换——如果在应用程序中使用updateStateByKey或reduceByKeyAndWindow(带有逆函数)，那么必须提供检查点目录来允许定期的RDD检查。
  * 从运行应用程序的驱动程序的故障中恢复——使用元数据检查点来恢复进度信息。

注意，没有上述有状态转换的简单流应用程序可以在不启用检查点的情况下运行。在这种情况下，从驱动程序故障中恢复也是部分的(一些接收到但未处理的数据可能丢失)。许多以这种方式运行Spark流应用程序通常是可以接受的。对非hadoop环境的支持有望在未来得到改善。

### 如何配置检查点(How to configure Checkpointing)

检查点可以通过在一个容错的、可靠的文件系统(如HDFS、S3等)中设置一个目录来启用，检查点信息将保存到该目录中。这是通过使用streamingContext.checkpoint(checkpointDirectory)实现的。这将允许您使用前面提到的有状态转换。此外，如果希望应用程序从驱动程序故障中恢复，应该重写流应用程序，使其具有以下行为。
 * 当程序第一次启动时，它将创建一个新的StreamingContext，设置所有的streams，然后调用start()。
 * 当程序在失败后重新启动时，它将从检查点目录中的检查点数据重新创建一个StreamingContext。

通过使用StreamingContext.getOrCreate可以简化此行为。它的用法如下。进入翻译页面
 ```scala
 // Function to create and setup a new StreamingContext
def functionToCreateContext(): StreamingContext = {
  val ssc = new StreamingContext(...)   // new context
  val lines = ssc.socketTextStream(...) // create DStreams
  ...
  ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
  ssc
}

// Get StreamingContext from checkpoint data or create a new one
val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

// Do additional setup on context that needs to be done,
// irrespective of whether it is being started or restarted
context. ...

// Start the context
context.start()
context.awaitTermination()

```

如果存在checkpointDirectory，那么将从检查点数据重新创建上下文。如果该目录不存在(即，然后调用functionToCreateContext函数创建新上下文并设置DStreams。参见Scala示例[RecoverableNetworkWordCount](https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala。此示例将网络数据的单词计数附加到文件中。

除了使用getOrCreate之外，还需要确保驱动程序进程在失败时自动重新启动。这只能由用于运行应用程序的部署基础设施来完成。这将在[部署](https://spark.apache.org/docs/latest/streaming-programming-guide.html#deploying-applications)一节中进一步讨论。

注意，RDDs的检查点会增加将数据保存到可靠存储的成本。这可能会导致RDDs被检查点的那些批次的处理时间增加。因此，需要仔细设置检查点的间隔。在小批量情况下(比如1秒)，每批检查可能会显著降低操作吞吐量。相反，检查点太少会导致沿袭和任务大小增长，这可能会产生有害的影响。对于需要RDD检查点的有状态转换，默认间隔是至少10秒的批处理间隔的倍数。可以使用dstream.checkpoint(checkpointInterval)设置它。通常，一个DStream的5 - 10个滑动间隔的检查点间隔是一个很好的设置。

## 累加器、广播变量和检查点(Accumulators, Broadcast Variables, and Checkpoints)

无法从Spark Streaming中的检查点恢复累加器和广播变量。如果启用了检查点并同时使用累加器或广播变量，则必须为累加器和广播变量创建延迟实例化的单例实例，以便在驱动程序失败重新启动后重新实例化它们。如下面的例子所示。
```Scala
object WordBlacklist {

  @volatile private var instance: Broadcast[Seq[String]] = null

  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordBlacklist = Seq("a", "b", "c")
          instance = sc.broadcast(wordBlacklist)
        }
      }
    }
    instance
  }
}

object DroppedWordsCounter {

  @volatile private var instance: LongAccumulator = null

  def getInstance(sc: SparkContext): LongAccumulator = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          instance = sc.longAccumulator("WordsInBlacklistCounter")
        }
      }
    }
    instance
  }
}

wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) =>
  // Get or register the blacklist Broadcast
  val blacklist = WordBlacklist.getInstance(rdd.sparkContext)
  // Get or register the droppedWordsCounter Accumulator
  val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
  // Use blacklist to drop words and use droppedWordsCounter to count them
  val counts = rdd.filter { case (word, count) =>
    if (blacklist.value.contains(word)) {
      droppedWordsCounter.add(count)
      false
    } else {
      true
    }
  }.collect().mkString("[", ", ", "]")
  val output = "Counts at time " + time + " " + counts
})
```
查看完整的[源代码](https://github.com/apache/spark/blob/v2.4.4/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala)。


## 部署应用(Deploying Applications)

本节讨论部署Spark流应用程序的步骤。

### 要求
 * 具有集群管理器的集群——这是任何Spark应用程序的一般需求，并在[部署指南](https://spark.apache.org/docs/latest/cluster-overview.html)中详细讨论。
 * 应用打成jar包——须将流应用程序编译到JAR包中。如果使用[Spark -submit](https://spark.apache.org/docs/latest/submitting-applications.html)启动应用程序，则不需要在JAR中提供Spark和Spark流所对应的jar包。但是，如果您的应用程序使用高级的[数据源](https://spark.apache.org/docs/latest/streaming-programming-guide.html#advanced-sources)(例如Kafka、Flume)，那么您必须将所依赖的jar包及其依赖项打包到用于部署应用程序的JAR中。例如，使用KafkaUtils的应用程序必须在应用程序JAR中包含spark-streaming-kafka-0-10_2.12及其所有传递依赖项。
 * 为执行器配置足够的内存-- 由于接收到的数据必须存储在内存中，所以必须为执行器配置足够的内存来保存接收到的数据。注意，如果您正在执行10分钟的窗口操作，则系统必须在内存中保留至少10分钟的数据。因此，应用程序的内存需求取决于其中使用的操作。
  * 配置检查点——如果流应用程序需要它，那么必须将Hadoop API兼容容错存储中的一个目录(如HDFS、S3等)配置为检查点目录，并以检查点信息可用于故障恢复的方式编写流应用程序。有关更多细节，请参见检查点部分。
  * 配置应用程序驱动程序的自动重启——为了从驱动程序失败中自动恢复，用于运行流应用程序的部署基础设施必须监视驱动程序进程，并在驱动程序失败时重新启动驱动程序。不同的集群管理器有不同的工具来实现这一点。

    - Spark Standalone —可以提交一个Spark应用程序驱动程序在Spark Standalone集群中运行(请参阅[集群部署模式](https://spark.apache.org/docs/latest/spark-standalone.html#launching-spark-applications))，也就是说，应用程序驱动程序本身在一个工作节点上运行。此外，可以指示独立集群管理器监视驱动程序，并在驱动程序由于非零退出码或运行驱动程序的节点失败而失败时重新启动它。有关更多详细信息，请参阅【Spark standlone指南](https://spark.apache.org/docs/latest/spark-standalone.html)中的集群模式和监督。
    - YARN -- Yarn支持类似的自动重新启动应用程序的机制。请参阅YARN文档的更多细节。
    - Mesos - [Marathon](https://github.com/mesosphere/marathon)已经被用来实现Mesos。
 * 配置写前日志——从Spark 1.2开始，我们就引入了写前日志，以实现强大的容错保证。如果启用，则从接收器接收到的所有数据都将写入配置检查点目录中的写前日志。这可以防止在驱动程序恢复时丢失数据，从而确保零数据丢失(在[容错语义](https://spark.apache.org/docs/latest/streaming-programming-guide.html#fault-tolerance-semantics)一节中详细讨论)。
这可以通过设置配置参数spark.stream .receiver. writeaheadlog来启用，开启为true。然而，这些更强的容错可能以单个接收器的接收吞吐量为代价。这可以通过[并行运行](https://spark.apache.org/docs/latest/streaming-programming-guide.html#level-of-parallelism-in-data-receiving)更多的接收器来纠正，以增加总吞吐量。此外，建议在启用写前日志时禁用Spark中接收数据的复制，因为日志已经存储在复制的存储系统中。这可以通过将输入流的存储级别设置为StorageLevel.MEMORY_AND_DISK_SER来实现。在使用S3(或任何不支持刷新的文件系统)写前日志时，请记住启用spark.stream.driver. writeaheadlog.closeFileAfterWrite和 spark.streaming.receiver.writeAheadLog.closeFileAfterWrite。有关详细信息，请参阅[Spark Streaming](https://spark.apache.org/docs/latest/configuration.html#spark-streaming)配置。请注意，当启用I/O加密时，Spark不会加密写入write-ahead日志的数据。如果需要对写前日志数据进行加密，则应该将其存储在本地支持加密的文件系统中。
 * 设置最大接收速率—如果集群资源不够大，流应用程序无法像接收数据一样快速处理数据，则可以通过设置记录/秒的最大速率限制来限制接收方的速率。参见配置参数spark.stream.receiver.maxRate的配置接收器和spark.streaming.kafka.maxRatePerPartitionmaxRatePerPartition直接配置kafka。在Spark 1.5中，我们引入了一个称为回压的特性，它消除了设置速率限制的需要，因为Spark Streaming自动计算速率限制，并在处理条件发生变化时动态调整它们。通过设置配置参数spark.stream.backpressure可以启用它的回压。启用为true。

## 升级应用程序代码(Upgrading Application Code)

如果正在运行的Spark STreaming应用程序需要使用新的应用程序代码进行升级，那么有两种可能的机制。
 * 升级后的Spark Streaming应用程序将与现有应用程序并行启动和运行。一旦新服务器(接收与旧服务器相同的数据)被预热并准备好进入黄金时间，旧服务器就可以被关闭。注意，对于支持将数据发送到两个目的地(即、早期和升级的应用程序)。
 * 现有应用程序被正常关闭(有关优美的关闭选项，请参阅[StreamingContext.stop(…)](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.StreamingContext)或[JavaStreamingContext.stop(…)](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html))确保已接收的数据在关机前已完全处理完毕。然后可以启动升级的应用程序，它将从先前的应用程序停止的地方开始处理。请注意，这只能在支持源端缓冲的输入源(如Kafka和Flume)中完成，因为需要在前一个应用程序宕机而升级的应用程序尚未启动时对数据进行缓冲。无法从升级前代码的早期检查点信息重新启动。检查点信息本质上包含序列化的Scala/Java/Python对象，尝试使用新的、修改过的类来反序列化对象可能会导致错误。在这种情况下，可以使用不同的检查点目录启动升级后的应用程序，也可以删除以前的检查点目录。

## 监视应用程序(Monitoring Applications)
除了Spark的[监控功能](https://spark.apache.org/docs/latest/monitoring.html)，还有一些特定于Spark流的附加功能。当使用StreamingContext时，Spark web UI会显示一个附加的流选项卡，其中显示关于正在运行的接收方(接收方是否活动、接收到的记录数量、接收方错误等)和完成的批(批处理时间、队列延迟等)的统计信息。这可以用来监视流应用程序的进度。

web UI中的以下两个指标特别重要:
 * 处理时间——处理每批数据的时间。
 * 调度延迟——批处理在队列中等待前一批处理完成的时间。

如果批处理时间始终大于批处理间隔和/或队列延迟不断增加，则表明系统无法像生成批处理那样快速地处理它们，并且正在处理落后。在这种情况下，可以考虑[减少](https://spark.apache.org/docs/latest/streaming-programming-guide.html#reducing-the-batch-processing-times)批处理时间。

Spark Streaming程序的进程也可以使用StreamingListener接口进行监视，该接口允许您获得接收方状态和处理时间。请注意，这是一个开发人员API，它可能会得到改进(即更多信息能被获取)在未来。
