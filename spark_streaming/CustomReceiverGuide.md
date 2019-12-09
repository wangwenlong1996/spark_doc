# Spark Streaming自定义接收器(Spark Streaming Custom Receivers)

Spark Streaming除了它内置支持的数据源之外(也就是说，除了Flume、Kafka、Kinesis、文件、socket等)，也可以从其他任何数据源接收流数据。 这要求开发人员实现一个为接收来自相关数据源的数据而定制的接收器。本指南介绍了实现自定义接收器并在Spark流应用程序中使用它的过程。注意，自定义接收器可以用Scala或Java实现。

## 实现自定义接收器(Implementing a Custom Receiver)
首先实现一个接收器([Scala doc](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.receiver.Receiver)、[Java doc](https://spark.apache.org/docs/latest/api/java/org/apache/spark/streaming/receiver/Receiver.html))。自定义接收器必须通过实现两个方法来扩展这个抽象类。
 * `onStart()`:开始接收数据要做的事情。
 * `onStop()`:停止接收数据要做的事情。

onStart()和onStop()不能无限阻塞。通常，onStart()将启动负责接收数据的线程，而onStop()将确保这些接收数据的线程被停止。接收线程也可以使用isStopped()方法来检查它们是否应该停止接收数据。

一旦接收到数据，就可以通过调用store(data)将数据存储在Spark中，这是Receiver类提供的方法。 store()有多种形式，可以一次存储接收到的数据记录，也可以作为对象/序列化字节的整个集合。注意，用于实现接收器的store()的风格会影响其可靠性和容错语义。稍后将对此进行更详细的讨论。

接收线程中的任何异常都应该被捕获并正确处理，以避免接收方的无声故障。restart(<exception>)将通过异步调用onStop()和延迟后调用onStart()来重新启动接收器。stop(<exception>)将调用onStop()并终止接收方。此外，reportError(<error>)在不停止/重新启动接收器的情况下向驱动程序报告错误消息(在日志和UI中可见)。下面是一个通过套接字接收文本流的自定义接收器。它将文本流中“\n”分隔的行作为记录，并使用Spark存储它们。如果接收线程在连接或接收时发生任何错误，则重新启动接收方，再次尝试连接。
```Scala
class CustomReceiver(host: String, port: Int)
  extends Receiver[String](StorageLevel.MEMORY_AND_DISK_2) with Logging {

  def onStart() {
    // Start the thread that receives data over a connection
    new Thread("Socket Receiver") {
      override def run() { receive() }
    }.start()
  }

  def onStop() {
    // There is nothing much to do as the thread calling receive()
    // is designed to stop by itself if isStopped() returns false
  }

  /** Create a socket connection and receive data until receiver is stopped */
  private def receive() {
    var socket: Socket = null
    var userInput: String = null
    try {
      // Connect to host:port
      socket = new Socket(host, port)

      // Until stopped or connection broken continue reading
      val reader = new BufferedReader(
        new InputStreamReader(socket.getInputStream(), StandardCharsets.UTF_8))
      userInput = reader.readLine()
      while(!isStopped && userInput != null) {
        store(userInput)
        userInput = reader.readLine()
      }
      reader.close()
      socket.close()

      // Restart in an attempt to connect again when server is active again
      restart("Trying to connect again")
    } catch {
      case e: java.net.ConnectException =>
        // restart if could not connect to server
        restart("Error connecting to " + host + ":" + port, e)
      case t: Throwable =>
        // restart if there is any other error
        restart("Error receiving data", t)
    }
  }
}
```
## 在Spark Streaming应用程序中使用自定义接收器(Using the custom receiver in a Spark Streaming application)

通过使用streamingContext，可以在Spark STreaming应用程序中使用自定义接收器。receiverStream(<instance of custom receiver>)。这将创建一个输入DStream的实例,它使用自定义接收器接受数据，如下所示:
```
// Assuming ssc is the StreamingContext
val customReceiverStream = ssc.receiverStream(new CustomReceiver(host, port))
val words = customReceiverStream.flatMap(_.split(" "))
```
完整的源代码在示例[CustomReceiver.scala](https://github.com/apache/spark/blob/v2.4.4/examples/src/main/scala/org/apache/spark/examples/streaming/CustomReceiver.scala)中。

## 接收机的可靠性(Receiver Reliability)
如Spark Streaming 编程指南中简要讨论的，基于可靠性和容错语义，有两种接收器。
 1. 可靠的接收方——对于允许发送数据被确认的可靠源，可靠的接收方正确地向源确认数据已被接收并可靠地存储在Spark中(即已成功复制)。通常，实现此接收器需要仔细考虑源确认的语义。
 2. 不可靠的接收方——不可靠的接收方不向源发送确认。这可以用于不支持确认的源，甚至可以用于不希望或不需要考虑acknowle复杂性的可靠源。

 要实现可靠的接收器，必须使用store(multiple-records)来存储数据。这种类型的存储是一个阻塞调用，它只在所有给定的记录都存储在Spark中之后才返回。 如果接收方配置的存储级别使用复本(默认启用)，则此调用在复制完成后返回。因此，它确保数据被可靠地存储，并且接收者现在可以适当地确认源。这可以确保在复制数据的过程中不会丢失任何数据——缓存的数据将不会被确认，因此稍后将被源重新发送。不可靠的接收方不必实现这些逻辑。它可以简单地从源接收记录，然后使用store(单记录)一次插入一条记录。虽然没有得到存储(多记录)的可靠性保证，但具有以下优点:
  * 系统负责将数据分块成适当大小的块(在Spark流编程指南中查找块间隔)。
  * 如果指定了速率限制，则系统负责控制接收速率。
  * 由于这两个原因，不可靠的接收器比可靠的接收器更容易实现。

下表总结了两种类型的接收器的特点：
接收器类型| 特点
-|-
不可靠的接收器|简单的实现。系统负责块生成和速率控制。没有容错保证，在接收端可能丢失数据失败
可靠的接收器|强大的容错保证，能保证零数据丢失。由接收方实现处理的块生成和速率控制。实现的复杂性取决于源的确认机制。
