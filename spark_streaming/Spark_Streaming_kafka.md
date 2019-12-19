# Spark Streaming +Kafka 集成指南

Apache Kafka 是作为发布-订阅消息传递的重新思考，它是分布式、分区、复制提交日志服务的。在使用Spark开始集成之前，请仔细阅读Kafka文档。

Kafka项目在0.8和0.10版本之间引入了一个新的消费者API，因此有两个单独的相应的Spark流= Streaming 应用包。请为您的代理(中间件)和所需功能选择正确的软件包；请注意，0.8集成与以后的0.9和0.10代理兼容，但0.10集成与早期的代理不兼容。
**注意:从Spark 2.3.0开始就不支持Kafka 0.8了。**

version|[spark-streaming-kafka-0-8](https://spark.apache.org/docs/latest/streaming-kafka-0-8-integration.html)|[spark-streaming-kafka-0-10](https://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)
-|-|-
代理版本|0.8.2.1 or higher|0.10.0 or higher
API成熟|弃用|稳定
语言支持|Scala, Java, Python|Scala, Java
接收器DStream|YES|NO
Direct DStream|YES|YES
SSL / TLS Support|NO|YES
Offset Commit API|NO|YES
Dynamic Topic Subscription|	No|YES
