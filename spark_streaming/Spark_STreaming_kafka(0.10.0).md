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
