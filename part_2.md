
[TOC]
# 部署应用(Deploying Applications)

本节讨论部署Spark流应用程序的步骤。
## 要求
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

# 升级应用程序代码(Upgrading Application Code)

如果正在运行的Spark STreaming应用程序需要使用新的应用程序代码进行升级，那么有两种可能的机制。
 * 升级后的Spark Streaming应用程序将与现有应用程序并行启动和运行。一旦新服务器(接收与旧服务器相同的数据)被预热并准备好进入黄金时间，旧服务器就可以被关闭。注意，对于支持将数据发送到两个目的地(即、早期和升级的应用程序)。
 * 现有应用程序被正常关闭(有关优美的关闭选项，请参阅[StreamingContext.stop(…)](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.StreamingContext)或[JavaStreamingContext.stop(…)](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html))确保已接收的数据在关机前已完全处理完毕。然后可以启动升级的应用程序，它将从先前的应用程序停止的地方开始处理。请注意，这只能在支持源端缓冲的输入源(如Kafka和Flume)中完成，因为需要在前一个应用程序宕机而升级的应用程序尚未启动时对数据进行缓冲。无法从升级前代码的早期检查点信息重新启动。检查点信息本质上包含序列化的Scala/Java/Python对象，尝试使用新的、修改过的类来反序列化对象可能会导致错误。在这种情况下，可以使用不同的检查点目录启动升级后的应用程序，也可以删除以前的检查点目录。

# 监视应用程序(Monitoring Applications)
除了Spark的[监控功能](https://spark.apache.org/docs/latest/monitoring.html)，还有一些特定于Spark流的附加功能。当使用StreamingContext时，Spark web UI会显示一个附加的流选项卡，其中显示关于正在运行的接收方(接收方是否活动、接收到的记录数量、接收方错误等)和完成的批(批处理时间、队列延迟等)的统计信息。这可以用来监视流应用程序的进度。

web UI中的以下两个指标特别重要:
 * 处理时间——处理每批数据的时间。
 * 调度延迟——批处理在队列中等待前一批处理完成的时间。

如果批处理时间始终大于批处理间隔和/或队列延迟不断增加，则表明系统无法像生成批处理那样快速地处理它们，并且正在处理落后。在这种情况下，可以考虑[减少](https://spark.apache.org/docs/latest/streaming-programming-guide.html#reducing-the-batch-processing-times)批处理时间。

Spark Streaming程序的进程也可以使用StreamingListener接口进行监视，该接口允许您获得接收方状态和处理时间。请注意，这是一个开发人员API，它可能会得到改进(即更多信息能被获取)在未来。

# 性能调优(Performance Tuning)

要在集群上的Spark Streaming应用程序中获得最佳性能，需要进行一些调整。这些已在[调优指南](https://spark.apache.org/docs/latest/tuning.html)中详细讨论。本节重点介绍一些最重要的内容。
## 数据接收的并行度(Level of Parallelism in Data Receiving)
通过网络接收数据(如Kafka、Flume、socket等)需要将数据反序列化并存储在Spark中。如果数据接收成为系统中的瓶颈，那么可以考虑将数据接收并行化。注意，每个输入DStream创建一个接收方(运行在工作机器上)，接收一个数据流。因此，通过创建多个输入数据流并配置它们以从源接收不同的数据流分区，可以实现接收多个数据流。例如，一个接收两个数据主题的Kafka输入DStream可以分成两个Kafka输入流，每个输入流只接收一个主题。这将运行两个接收器，允许并行接收数据，从而提高总体吞吐量。可以将多个DStream合并在一起来创建单个DStream。然后，应用于单个输入DStream上的转换可以应用于统一的流。这是这样做的。
```Scala
val numStreams = 5
val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
val unifiedStream = streamingContext.union(kafkaStreams)
unifiedStream.print()
```
另一个需要考虑的参数是接收器的block interva，它是由[配置参数](https://spark.apache.org/docs/latest/configuration.html#spark-streaming)spark.stream.blockinterval决定的。对于大多数接收方来说，接收到的数据在存储到Spark内存之前会被合并成数据 block。每个批处理中的块的数量确定了在map转换中用于处理接收数据的任务数。每批每个接收方的任务数量是可预估的(batch interval / block interval)。例如，200 ms的block interval将每批2秒创建10个任务。如果任务的数量过低(即少于每台机器的内核数量)，那么它将是低效的，因为有可用的内核未用于处理数据。若要增加给定batch interval的任务数量，请减少block interval。但是，建议的 block interval最小值为50 ms，低于这个值可能会导致任务启动开销出现问题。

使用多个输入流/接收器接收数据的替代方法是显式地重新划分输入数据流(使用inputStream.repartition(<number of partitions>))。在进一步处理之前，它将接收到的数据重新划分分布到集群中指定数量的机器上。

对于 direct stream，请参阅[Spark Streaming+ Kafka集成指南](https://spark.apache.org/docs/latest/streaming-kafka-integration.html)

## 数据处理的并行度(Level of Parallelism in Data Processing)

如果在计算的任何阶段中使用的并行任务的数量都不够高，则集群资源可能没有得到充分利用。例如，对于诸如reduceByKey和reduceByKeyAndWindow之类的分布式归约操作，并行任务的默认数量由spark.default.parallelism[配置属性](https://spark.apache.org/docs/latest/configuration.html#spark-properties)控制。您可以将并行度级别作为参数传递(请参阅[PairDStreamFunctions](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions)文档)，或者设置spark.default.parallelism配置属性来更改默认值。

## 数据序列化(Data Serialization)

通过调整序列化格式，可以减少数据序列化的开销。对于流，有两种类型的数据正在被序列化。
  * **输入数据**： 默认情况下，通过接收器接收到的输入数据通过StorageLevel.MEMORY_AND_DISK_SER_2存储在执行器的内存中。也就是说，数据被序列化为字节以减少GC开销，并被复制以容忍执行器故障。此外，数据首先保存在内存中，只有在内存不足以容纳流计算所需的所有输入数据时才会溢出到磁盘。这种序列化显然有开销——接收方必须反序列化接收到的数据，然后使用Spark的序列化格式重新序列化它。
  * **持久化由流操作生成的RDDs**：持久化由流操作生成的RDDs。例如，窗口操作将数据保存在内存中，因为它们将被多次处理。然而，不像Spark Core默认的StorageLevel。通过流计算生成的持久化的RDDs使用StorageLevel持久化。默认是MEMORY_ONLY_SER(即序列化)，以最小化GC开销。

在这两种情况下，使用Kryo序列化可以减少CPU和内存开销。有关更多细节，请参阅Spark[调优指南](https://spark.apache.org/docs/latest/tuning.html#data-serialization)
。对于Kryo，可以考虑注册自定义类，并禁用对象引用跟踪(请参阅配置指南中与Kryo相关的[配置](https://spark.apache.org/docs/latest/configuration.html#compression-and-serialization))。

在需要为流应用程序保留的数据量不大的特定情况下，可以将数据(两种类型)作为反序列化对象持久存储（数据存储不序列化），而不会导致过多的GC开销。例如，如果您正在使用几秒钟的批处理间隔，并且没有窗口操作，那么您可以通过显式地相应地设置存储级别来尝试禁用持久数据中的序列化。这将减少由于序列化而导致的CPU开销，从而在没有太多GC开销的情况下提高性能。

## 任务启动开销(Task Launching Overheads)

如果每秒启动的任务数量很高(比如每秒50个或更多)，那么向从属服务器发送任务的开销可能会很大，并且很难实现次秒延迟。可以通过以下改变来减少开销:
 * **执行模式**: 在独立模式或粗粒度Mesos模式下运行Spark比细粒度[Mesos模式](https://spark.apache.org/docs/latest/running-on-mesos.html)下的任务启动时间更长。

这些更改可能会将批处理时间减少100毫秒，从而使次秒级的批大小成为可能。

## 设置正确的批处理间隔(Setting the Right Batch Interval)

要使运行在集群上的Spark Streaming应用程序稳定，系统应该能够处理接收到的数据。换句话说，处理批量数据的速度应该与生成数据的速度一样快。通过[监视](https://spark.apache.org/docs/latest/streaming-programming-guide.html#monitoring-applications)流web UI中的处理时间可以发现应用程序是否如此，其中批处理时间应该小于批处理间隔。
根据流计算的性质，所使用的批处理间隔可能对应用程序在一组固定的集群资源上能够维持的数据速率产生重大影响。例如，让我们考虑前面的WordCountNetwork示例。对于特定的数据速率，系统可以每2秒报告一次字数计数(即，批处理间隔为2秒)，但不是每500毫秒一次。因此需要设置批处理间隔，以便能够维持生产中的预期数据速率。

为应用程序确定正确的批大小的一个好方法是使用保守的批处理间隔(例如，5-10秒)和较低的数据速率对其进行测试。要验证系统是否能够跟上数据速率，可以检查每个处理批所经历的端到端延迟的值(在Spark驱动程序log4j日志中查找“总延迟”，或者使用StreamingListener接口)。如果延迟保持与批处理大小相当，则系统是稳定的。否则，如果延迟持续增加，则意味着系统无法跟上，因此不稳定。一旦有了稳定配置的想法，就可以尝试增加数据速率和/或减少批处理大小。请注意，由于临时数据速率的增加而导致的暂时延迟的增加可能是正确的，只要延迟减少到一个较低的值(即，小于批处理大小)。

## 内存调优

调优Spark应用程序的内存使用和GC行为在[调优指南](https://spark.apache.org/docs/latest/tuning.html#memory-tuning)中有详细的讨论。强烈建议你读一读。在本节中，我们将讨论一些特定于Spark Streaming应用程序上下文的调优参数。

Spark STreaming应用程序所需的集群内存量在很大程度上取决于所使用的transformations类型。例如，如果您想对前10分钟的数据使用窗口操作，那么您的集群应该有足够的内存来在内存中保存10分钟的数据。或者，如果您希望使用带有大量键的updateStateByKey，那么所需的内存将会很大。相反，如果您想要执行简单的map-filter-store操作，那么所需的内存将会很低。

一般来说，由于通过接收器接收到的数据是用StorageLevel.MEMORY_AND_DISK_SER_2，不能够用内存存储的数据将溢出到磁盘。这可能会降低流应用程序的性能，因此建议根据流应用程序的需要提供足够的内存。最好尝试在小范围内查看内存使用情况并进行相应的估计。

内存调优的另一个方面是垃圾收集。对于需要低延迟的流应用程序，JVM垃圾收集导致的大量暂停是不可取的。

有几个参数可以帮助你调优内存使用和GC开销:

 * **DStreams的持久性级别**： 正如前面在数据序列化一节中提到的，输入数据和RDDs在默认情况下是作为序列化字节持久化的。与反序列化持久性相比(直接保存在内存)，这减少了内存使用和GC开销。启用Kryo序列化进一步减少了序列化的大小和内存使用。通过压缩(参见Spark配置Spark .rdd.compress)可以进一步减少内存使用量，但要以CPU时间为代价。

  * **清除旧数据**： 默认情况下，由DStream转换生成的所有输入数据和持久的RDDs将被自动清除。Spark Streaming根据使用的转换决定何时清除数据。例如，如果您使用的是10分钟的窗口操作，那么Spark流将保留最后10分钟的数据，并主动丢弃旧的数据。通过设置streamingContext.remember，可以更长时间地保留数据(例如交互式地查询旧数据)。

  * **CMS垃圾收集器**: 强烈建议使用并发标记-清除GC，以保持与GC相关的暂停始终较低。尽管众所周知并发GC会降低系统的总体处理吞吐量，但仍然建议使用它来实现更一致的批处理时间。确保在驱动程序(在Spark-submit中使用--driver-java-options)和执行器(使用Spark配置Spark.executor.extrajavaoptions)上都设置了CMS GC。
  * **其它建议**: 为了进一步减少GC开销，这里还有一些技巧可以尝试。

    - 使用OFF_HEAP存储级别持久化RDDs。请参阅[Spark编程指南](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)。
    - 使用更小堆大小的执行器。这将减少每个JVM堆中的GC压力。

**要记住的要点**:
 * DStream与单个接收器相关联。为了获得读并行性，需要创建多个接收器，即多个DStreams。接收器在执行器中运行。它占据一个核。确保在接收槽被预定后有足够的内核用于处理数据，spark.cores.max参数应该包含接收槽。接收者以循环方式分配给执行者。

 * 当从流源接收数据时，接收器创建数据块。每隔一毫秒就会产生一个新的数据块。在batchInterval期间创建N个数据块，其中N = batchInterval/blockInterval。这些块由当前执行程序的块管理器分发给其他执行程序的块管理器。之后，运行在驱动程序上的网络输入跟踪器将被告知进一步处理的块位置。

 * 在驱动程序上为在batchInterval期间创建的块，创建了RDD。batchInterval期间生成的块是RDD的分区。每个分区都是spark中的一个任务。blockInterval== batchinterval将意味着创建一个单独的分区，并且可能在本地处理它。

 * 块上的映射任务在执行器(一个接收块，另一个复制块)中处理，不管块的间隔如何，除非出现非本地调度。拥有更大的blockinterval意味着更大的块。spark.locality.wait大的值增加了在本地节点上处理一个块的机会。需要在这两个参数之间找到平衡，以确保在本地处理较大的块。

 * 您可以通过调用inputDstream.repartition(n)来定义分区的数量，而不是依赖于batchInterval和blockInterval。这将随机重组RDD中的数据，以创建n个分区。是的，为了更好的并行性。不过代价是shuffle。RDD的处理由驱动程序的jobscheduler作为job调度。
 * 如果你有两个dstreams，就会形成两个RDDs，就会创建两个作业，它们会一个接一个地安排。为了避免这种情况，可以union两个dstreams。这将确保dstreams的两个rdd形成一个unionRDD。然后，这个unionRDD被视为单个作业。但是，RDDs的分区不受影响。
 * 如果批处理时间大于batchinterval，那么显然接收方的内存将开始填满，并最终抛出异常(很可能是BlockNotFoundException)。目前，没有办法暂停接收器。使用SparkConf配置spark.stream.receiver.maxRate，限制接受速率。
