# Spark Streaming + Kafka 集成指南(kafka代理节点版本0.10.0或者更高) (Spark Streaming + Kafka Integration Guide (Kafka broker version 0.10.0 or higher))

Kafka 0.10的Spark Streaming集成在设计上类似于0.8 [Direct Stream方法](https://spark.apache.org/docs/latest/streaming-kafka-0-8-integration.html#approach-2-direct-approach-no-receivers)。它提供简单的并行性，Kafka分区和Spark分区之间的1：1对应关系以及对偏移量和元数据的访问。但是，由于较新的集成使用了新的Kafka使用者API而不是简单的API，因此用法上存在显着差异。集成的此版本标记为实验性的，因此API可能会发生更改。

## 链接(Linking)
对于使用SBT / Maven项目定义的Scala / Java应用程序，将流式应用程序与以下依赖链接（有关更多信息，请参见主编程指南中的[链接](https://spark.apache.org/docs/latest/streaming-programming-guide.html#linking)部分）。
```maven
groupId = org.apache.spark
artifactId = spark-streaming-kafka-0-10_2.12
version = 2.4.4
```

不要手动添加对org.apache.kafka的依赖（例如kafka-clients）。spark-streaming-kafka-0-10已经具有适当的传递依赖性，并且不同的版本产生问题可能难以诊断，以至于不兼容。

## 创建直接流(Creating a Direct Stream)

请注意，导入的名称空间包括版本org.apache.spark.streaming.kafka010

```scala
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.kafka010.LocationStrategies.PreferConsistent
import org.apache.spark.streaming.kafka010.ConsumerStrategies.Subscribe

val kafkaParams = Map[String, Object](
  "bootstrap.servers" -> "localhost:9092,anotherhost:9092",
  "key.deserializer" -> classOf[StringDeserializer],
  "value.deserializer" -> classOf[StringDeserializer],
  "group.id" -> "use_a_separate_group_id_for_each_stream",
  "auto.offset.reset" -> "latest",
  "enable.auto.commit" -> (false: java.lang.Boolean)
)

val topics = Array("topicA", "topicB")
val stream = KafkaUtils.createDirectStream[String, String](
  streamingContext,
  PreferConsistent,
  Subscribe[String, String](topics, kafkaParams)
)

stream.map(record => (record.key, record.value))
```

流中的每个项目都是一个[ConsumerRecord](http://kafka.apache.org/0100/javadoc/org/apache/kafka/clients/consumer/ConsumerRecord.html)。

有关可能的kafkaParams，请参阅[Kafka使用者配置文档](http://kafka.apache.org/documentation.html#newconsumerconfigs)。如果您的Spark批处理持续时间大于默认的Kafka心跳会话超时（30秒），适当增加heartbeat.interval.ms和session.timeout.ms。对于大于5分钟的批处理，这将需要更改代理上的group.max.session.timeout.ms。请注意，示例将enable.auto.commit设置为false，有关讨论，请参见下面的[存储偏移](https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html#storing-offsets)。

## 区位策略(LocationStrategies)

新的Kafka消费者API将消息预取到缓冲区中。因此，出于性能方面的考虑，Spark集成将缓存的消费者保留在执行器中(而不是为每个批处理重新创建它们)，并且更喜欢将分区安排在拥有适当消费者的主机位置上，这一点非常重要。

在大多数情况下，你应该使用LocationStrategies.PreferConsistent，如上面所示。如果您的执行器与Kafka代理位于同一主机上，请使用PreferBrokers，它将更愿意为该分区安排在Kafka领导者上的分区。最后，如果分区之间的负载有明显的倾斜，那么使用PreferFixed。这允许您指定分区到主机的显式映射(任何未指定的分区将使用一致的位置)。

消费者的缓存的默认最大值为64。如果您希望处理超过（64 * 执行器数）个Kafka分区，则可以通过spark.streaming.kafka.consumer.cache.maxCapacity更改此设置。

如果您想要禁用Kafka消费者的缓存，您可以设置spark.stream.Kafka.consumer.cache为false。

缓存由topicpartition和group.id作为键，所以使用一个单独的group.id调用createDirectStream。

## 消费者策略(ConsumerStrategies)

新的Kafka消费者API有许多不同的方式来指定主题，其中一些需要大量的对象实例化后的设置。消费者策略提供了一个抽象，允许Spark即使在从检查点重新启动之后也可以获得正确配置的消费者。

如上所示，ConsumerStrategies.Subscribe允许您订阅固定的主题集合。SubscribePattern允许您使用正则表达式来指定感兴趣的主题。注意，与0.8集成不同，使用Subscribe或SubscribePattern应该在运行流期间响应添加分区。最后，Assign允许您指定一个固定的分区集合。这三种策略都有重载的构造函数，允许您指定特定分区的起始偏移量。

如果您有特定的消费者设置需求，而上面的选项不能满足这些需求，那么ConsumerStrategy是一个可以扩展的公共类。

## 创建RDD(Creating an RDD)

如果您有通过为定义的偏移量范围创建一个RDD，是一个更适合批处理的用例。

```scala
val offsetRanges = Array(
  // topic, partition, inclusive starting offset, exclusive ending offset
  OffsetRange("test", 0, 0, 100),
  OffsetRange("test", 1, 0, 100)
)

val rdd = KafkaUtils.createRDD[String, String](sparkContext, kafkaParams, offsetRanges, PreferConsistent)
```

请注意，您不能使用preferbroker，因为没有流，就没有一个驱动端使用者为您自动查找代理元数据。如果需要，可以在自己的元数据查找中使用PreferFixed。

## 获取偏移量(Obtaining Offsets)

```scala
stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  rdd.foreachPartition { iter =>
    val o: OffsetRange = offsetRanges(TaskContext.get.partitionId)
    println(s"${o.topic} ${o.partition} ${o.fromOffset} ${o.untilOffset}")
  }
}
```
注意，对HasOffsetRanges的类型转换只有在对createDirectStream的结果调用的第一个方法中完成时才会成功，而不是在随后的方法链中。请注意，RDD分区和Kafka分区之间的一对一映射不会在任何洗牌或重分区的方法(例如reduceByKey()或window())之后仍然存在。

## 存储偏移量(Storing Offsets)

失败情况下的Kafka交付语义取决于如何以及何时存储偏移量。Spark输出操作至少一次。因此，如果您想要只有一次语义，您必须在幂等的输出之后存储偏移量。通过这种集成，您可以按照增加可靠性（和代码复杂度）的顺序使用3个选项来存储偏移量。

### 检查点(Checkpoints)

如果启用了Spark检查点，则偏移量将存储在检查点中。这很容易实现，但也有缺点。您的输出操作必须是幂等的，因为您将获得重复的输出。事务是不能选择的。此外，如果应用程序代码发生了更改，则无法从检查点恢复。对于计划中的升级，您可以通过在运行旧代码的同时运行新代码来缓解这种情况(因为无论如何输出都需要是幂等的，它们不应该冲突)。但是对于需要更改代码的计划外故障，除非您有其他方法来识别已知的良好起始偏移量，否则您将丢失数据。

### Kafka (Kafka itself)

Kafka有一个偏移量提交API，它将偏移量存储在一个特殊的Kafka主题中。默认情况下，新使用者将定期自动提交偏移量。这几乎肯定不是您想要的，因为由使用者成功轮询的消息可能尚未导致Spark输出操作，从而导致语义未定义。这就是为什么上面的流示例将“enable.auto.commit”设置为false。但是，您可以使用commitAsync API在知道您的输出已被存储后将偏移量提交给Kafka。与检查点相比，Kafka的优点是无论应用程序代码如何更改，它都是一个持久的存储。然而，Kafka不是事务性的，所以输出必须是幂等的。

```scala
stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

  // some time later, after outputs have completed
  stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)
}
```
与HasOffsetRanges一样，只有在createDirectStream的结果上调用时，转换到CanCommitOffsets才会成功，而不是在转换之后。commitAsync调用是线程安全的，但如果需要有意义的语义，则必须在输出之后执行。

### 自己存储数据(Your own data store)

对于支持事务的数据存储，将偏移量保存在与结果相同的事务中可以使两者保持同步，即使在出现故障的情况下也是如此。如果您在检测重复或跳过的偏移量范围时很谨慎，请回滚事务，以防止重复或丢失的消息影响结果。这就给出了完全一次的语义。甚至对于聚合产生的输出也可以使用这种策略，因为聚合通常很难实现幂等性。

```scala
// The details depend on your data store, but the general idea looks like this

// begin from the the offsets committed to the database
val fromOffsets = selectOffsetsFromYourDatabase.map { resultSet =>
  new TopicPartition(resultSet.string("topic"), resultSet.int("partition")) -> resultSet.long("offset")
}.toMap

val stream = KafkaUtils.createDirectStream[String, String](
  streamingContext,
  PreferConsistent,
  Assign[String, String](fromOffsets.keys.toList, kafkaParams, fromOffsets)
)

stream.foreachRDD { rdd =>
  val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

  val results = yourCalculation(rdd)

  // begin your transaction

  // update results
  // update offsets where the end of existing offsets matches the beginning of this batch of offsets
  // assert that offsets were updated correctly

  // end your transaction
}
```

## SSL/TLS

新的Kafka使用者[支持SSL](http://kafka.apache.org/documentation.html#security_ssl)。要启用它，请在传递到createDirectStream / createRDD之前设置kafkaParams。注意，这只适用于Spark和Kafka代理之间的通信;您仍然需要单独负责保护Spark节点间通信。

```scala
val kafkaParams = Map[String, Object](
  // the usual params, make sure to change the port in bootstrap.servers if 9092 is not TLS
  "security.protocol" -> "SSL",
  "ssl.truststore.location" -> "/some-directory/kafka.client.truststore.jks",
  "ssl.truststore.password" -> "test1234",
  "ssl.keystore.location" -> "/some-directory/kafka.client.keystore.jks",
  "ssl.keystore.password" -> "test1234",
  "ssl.key.password" -> "test1234"
)
```

## 部署(Deploying)

对于任何Spark应用程序，Spark -submit都用于启动您的应用程序。对于Scala和Java应用程序，如果您使用SBT或Maven进行项目管理，那么可以将spark-streaming-kafka-0-10_2.12及其依赖项打包到应用程序JAR中。确保Spark -core_2.12和Spark -streaming_2.12被标记为provided的依赖项，因为它们已经在Spark安装中存在。然后使用spark-submit启动应用程序(请参阅主编程指南中的[部署部分](https://spark.apache.org/docs/latest/streaming-programming-guide.html#deploying-applications))。
