#### 1 rpc介绍
spark作为一个分布式计算框架，必然涉及节点间通信， 节点间通信由rpc模块（ 远程过程调用， Remote Procedure Call）完成，rpc模块基于network_common模块实现。


#### 2 使用方式
相对于network_common，rpc使用起来相对简单，不需要实现&注册Callback、RpcHandler，不需要处理ByteBuf，只需要实现&注册RpcEndpoint

##### 2.1 client端代码<span id="myclient"></span>
```scala
package org.apache.spark.rpc

import org.apache.spark.{SecurityManager, SparkConf}
import org.apache.spark.rpc.netty.NettyRpcEnvFactory

import scala.concurrent.duration.Duration
import scala.concurrent.{Await, Future}


/**
  client使用流程：
       1.创建Rpc配置
       2.创建RpcEnv
       3.通过RpcEnv创建RpcEndpointRef
       4.通过RpcEndpointRef向远端Endpoint发送请求
       6.结果处理
  */
object MyRpcClient {

  def main(args: Array[String]): Unit = {
    val conf = new SparkConf()
    // Rpc配置
    val config = RpcEnvConfig(conf, "test", "localhost", "localhost", 0,
      new SecurityManager(conf), 0, true)
    // 创建RpcEnv
    val rpcEnv = new NettyRpcEnvFactory().create(config)
    // 创建到localhost:52333的RpcEndpointRef
    val endpointRef: RpcEndpointRef = rpcEnv.setupEndpointRef(RpcAddress("localhost", 52333), "my-rpc-endpoint")

    // 通过RpcEndpointRef发送类型为String的请求
    val resultStr:String  = endpointRef.askSync[String]("hello")
    println(s"resultStr: $resultStr")
    // 通过RpcEndpointRef发送类型为Int的请求
    val resultInt:Int  = endpointRef.askSync[Int](138)
    println(s"resultInt: $resultInt")
  }

}

```


##### 2.2 server端代码<span id="myserver"></span>

```scala
package org.apache.spark.rpc

import org.apache.spark.{SecurityManager, SparkConf, SparkException}
import org.apache.spark.rpc.netty.NettyRpcEnvFactory


/**
   server使用流程：
  1.创建Rpc配置，包含server address:port
  2.创建RpcEnv
  3.创建RpcEndpoint，并将RpcEndpoint注册到RpcEnv
  4.等待RpcEnv结束
  */
object MyRpcServer {

  def main(args: Array[String]): Unit = {

    val conf = new SparkConf()
    // Rpc配置
    val config = RpcEnvConfig(conf, "test", "localhost", "localhost", 52333,
      new SecurityManager(conf), 1, false)
    // 创建RpcEnv
    val rpcEnv = new NettyRpcEnvFactory().create(config)
    //注册RpcEndpoint，将RpcEndpoint交给RpcEnv管理
    rpcEnv.setupEndpoint("my-rpc-endpoint", new RpcEndpoint {

      override val rpcEnv: RpcEnv = rpcEnv

      override def onStart(): Unit = {
        // By default, do nothing.
      }

      override def onConnected(remoteAddress: RpcAddress): Unit = {
        println(s"connect from address: $remoteAddress")
      }

      //处理Rpc请求
      override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
        case msg: String => {
          //将请求msg+msg，然后返回
          context.reply(msg+msg)
        }

        case msg: Int => {
          //将请求msg+msg，然后返回
          context.reply(msg+msg)
        }

        case _ => context.sendFailure(new SparkException(self + " won't reply anything"))
      }

      override def onStop(): Unit = {
        // By default, do nothing.
      }

    })
    //等待RpcEnv结束
    rpcEnv.awaitTermination()
  }
```

##### 2.3 测试结果
- client端的输出

```
resultStr: hellohello
resultInt: 276

Process finished with exit code 0

```

- server端的输出

```
connect from address: 127.0.0.1:58190
```


#### 3  实现原理

rpc模块基于network-common实现了一套易于使用的通信框架，spark 1.6以前是使用akka作为rpc框架，spark 1.6以及1.6以后将akka移除了，rpc的组件跟akka组件非常相似，RpcEndpoint 对应akka Actor,
RpcEndpoint 对应akka Actorref, RpcEnv 对应akka ActorSystem

- RpcEndpoint是对rpc通信实体的抽象，所有运行在RpcEnv上的通信实体都应该继承RpcEndpoint

- RpcEndpointRef是指向RpcEndpoint，通过RpcEndpointRef可以向指向的RpcEndpoint发送请求

- RpcEnv管理RpcEndpoint, RpcEndpointRef

	- server端：RpcEnv负责管理RpcEndpoint的生命周期，并将请求路由到特定的RpcEndpoint

	- client端：RpcEnv可以获取RpcEndpoint引用，即RpcEnvRef

RpcEndpoint， RpcEndpointRef， RpcEnv定义的规范，具体实现要需要分析org.apache.spark.rpc.netty下的代码

#### 4  基于network-common的实现



Inbox： 存储RpcEndpoint的InboxMessage（in方向的消息， new java.util.LinkedList\[InboxMessage]()）

Outbox： 存储RpcEndpointRef的OutboxMessage,（out方向的消息， new java.util.LinkedList\[OutboxMessage]()）

Dispatcher: 消息分发器，将RequestMessage(in方向的消息)路由到正确的RpcEndpoint

NettyRpcEnv: RpcEnv的实现，管理RpcEndpoint， RpcEndpointRef，序列化/反序列化，network-common的TransportContext，TransportClient， TransportServer，RcpHandler等


##### 4.1 NettyRpcEnv初始化

###### 4.1.1 创建NettyRpcEnv的代码如下：

```scala
private[rpc] class NettyRpcEnvFactory extends RpcEnvFactory with Logging {

  def create(config: RpcEnvConfig): RpcEnv = {
    val sparkConf = config.conf
   
    // 通过反射创建序列化/反序列化器
    val javaSerializerInstance =
      new JavaSerializer(sparkConf).newInstance().asInstanceOf[JavaSerializerInstance]
    // 创建NettyRpcEnv 
    val nettyEnv =
      new NettyRpcEnv(sparkConf, javaSerializerInstance, config.advertiseAddress,
        config.securityManager, config.numUsableCores)
    //如果非client启动TransportServer    
    if (!config.clientMode) {
      val startNettyRpcEnv: Int => (NettyRpcEnv, Int) = { actualPort =>
        nettyEnv.startServer(config.bindAddress, actualPort)
        (nettyEnv, nettyEnv.address.port)
      }
      try {
        Utils.startServiceOnPort(config.port, startNettyRpcEnv, sparkConf, config.name)._1
      } catch {
        case NonFatal(e) =>
          nettyEnv.shutdown()
          throw e
      }
    }
    nettyEnv
  }
}
```
###### 4.1.2 NettyRpcEnv初始化过程

* 创建序列化器
* 创建NettyRpcEnv，这个步骤会初始化Dispatcher，NettyStreamManager，TransportContext，NettyRpcHandler，TransportClientFactory，注册RpcEndpointVerifier
* 如果非client模式，启动TransportServer


##### 4.2 Out方向(发送数据)处理流程

##### 4.2.1 Out方向的主要链路的代码


```
                  RpcEnv.setupEndpointRef 
                            |
                            |
                            V   
                   NettyRpcEndpointRef.ask  
                            |
                            |
                            V 
                     NettyRpcEnv.ask   
                            |
                            |
                            V   
  Dispatcher.postLocalMessage / NettyRpcEnv.postToOutbox 
                            |
                            |
                            V  
              OutboxMessage / Outbox.send      
                            |
                            |
                            V   
                    Outbox.drainOutbox      
```

##### 4.2.2 RpcEnv.setupEndpointRef 

RpcEnv.setupEndpointRef 主要的作用是创建EndpointRef，并检查目的Endpoint是否存在，通过RpcEndpointVerifier完成， 每个NettyRpcEnv在启动TransportServer时都会注册RpcEndpointVerifier


###### 4.2.2.1  注册RpcEndpointVerifier 代码如下：

```scala

  def startServer(bindAddress: String, port: Int): Unit = {
    val bootstraps: java.util.List[TransportServerBootstrap] =
      if (securityManager.isAuthenticationEnabled()) {
        java.util.Arrays.asList(new AuthServerBootstrap(transportConf, securityManager))
      } else {
        java.util.Collections.emptyList()
      }
    server = transportContext.createServer(bindAddress, port, bootstraps)
    //在Dispatcher注册RpcEndpointVerifier
    dispatcher.registerRpcEndpoint(
      RpcEndpointVerifier.NAME, new RpcEndpointVerifier(this, dispatcher))
  }
  
```


###### 4.2.2.2 创建EndpointRef代码如下：

创建EndpointRef的函数其实是调用了NettyRpcEnv.setupEndpointRefByURI函数进行创建

```scala
  def setupEndpointRef(address: RpcAddress, endpointName: String): RpcEndpointRef = {
    setupEndpointRefByURI(RpcEndpointAddress(address, endpointName).toString)
  }
  
```  

NettyRpcEnv.setupEndpointRefByURI函数代码如下：

```scala

  def asyncSetupEndpointRefByURI(uri: String): Future[RpcEndpointRef] = {
    val addr = RpcEndpointAddress(uri)
    val endpointRef = new NettyRpcEndpointRef(conf, addr, this)
    //创建verifier的EndpointRef
    val verifier = new NettyRpcEndpointRef(
      conf, RpcEndpointAddress(addr.rpcAddress, RpcEndpointVerifier.NAME), this)
    //向目标Endpoint发起Rpc请求，检查目标Endpoint是否存在
    verifier.ask[Boolean](RpcEndpointVerifier.CheckExistence(endpointRef.name)).flatMap { find =>
      if (find) {
        Future.successful(endpointRef)
      } else {
        Future.failed(new RpcEndpointNotFoundException(uri))
      }
    }(ThreadUtils.sameThread)
  }

```


RpcEndpointVerifier代码如下：
RpcEndpointVerifier的逻辑比较简单  （ps : RpcEndpointVerifier也是一种Rpc的请求，只不过 RpcEndpointVerifier作用在于检查目标Endpoint是否存在）


```scala
private[netty] class RpcEndpointVerifier(override val rpcEnv: RpcEnv, dispatcher: Dispatcher)
  extends RpcEndpoint {

  override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case RpcEndpointVerifier.CheckExistence(name) => context.reply(dispatcher.verify(name))
  }
}

private[netty] object RpcEndpointVerifier {
  val NAME = "endpoint-verifier"
  case class CheckExistence(name: String)
}
```

##### 4.2.3 NettyRpcEndpointRef.ask

NettyRpcEndpointRef.ask 将消息封装成RequestMessage，在调用NettyRpcEnv.ask， 具体代码如下：

```scala

def ask[T: ClassTag](message: Any, timeout: RpcTimeout): Future[T] = {
   nettyEnv.ask(new RequestMessage(nettyEnv.address, this, message), timeout)
}
  
```  

RequestMessage是一个比价重要的类，所有待发送的消息都会封装成RequestMessage，RequestMessage会将发送者地址、接收者的地址、消息序列化，所有待接收的消息都会反序列化发送者、接收者的地址、消息

```scala
private[netty] class RequestMessage(
    val senderAddress: RpcAddress,
    val receiver: NettyRpcEndpointRef,
    val content: Any) {

  //序列化
  def serialize(nettyEnv: NettyRpcEnv): ByteBuffer = {
    val bos = new ByteBufferOutputStream()
    val out = new DataOutputStream(bos)
    try {
      writeRpcAddress(out, senderAddress)
      writeRpcAddress(out, receiver.address)
      out.writeUTF(receiver.name)
      val s = nettyEnv.serializeStream(out)
      try {
        s.writeObject(content)
      } finally {
        s.close()
      }
    } finally {
      out.close()
    }
    bos.toByteBuffer
  }

  private def writeRpcAddress(out: DataOutputStream, rpcAddress: RpcAddress): Unit = {
    if (rpcAddress == null) {
      out.writeBoolean(false)
    } else {
      out.writeBoolean(true)
      out.writeUTF(rpcAddress.host)
      out.writeInt(rpcAddress.port)
    }
  }

  override def toString: String = s"RequestMessage($senderAddress, $receiver, $content)"
}

private[netty] object RequestMessage {

  private def readRpcAddress(in: DataInputStream): RpcAddress = {
    val hasRpcAddress = in.readBoolean()
    if (hasRpcAddress) {
      RpcAddress(in.readUTF(), in.readInt())
    } else {
      null
    }
  }
  // 反序列化
  def apply(nettyEnv: NettyRpcEnv, client: TransportClient, bytes: ByteBuffer): RequestMessage = {
    val bis = new ByteBufferInputStream(bytes)
    val in = new DataInputStream(bis)
    try {
      val senderAddress = readRpcAddress(in)
      val endpointAddress = RpcEndpointAddress(readRpcAddress(in), in.readUTF())
      val ref = new NettyRpcEndpointRef(nettyEnv.conf, endpointAddress, nettyEnv)
      ref.client = client
      new RequestMessage(
        senderAddress,
        ref,
        // The remaining bytes in `bytes` are the message content.
        nettyEnv.deserialize(client, bytes))
    } finally {
      in.close()
    }
  }
}
```
###### 4.2.4 NettyRpcEnv.ask

NettyRpcEnv.ask的主要工作，如果目标Endpoint等于发送Endpoint，通过Dispatcher.postLocalMessage将消息路由到向相应的Inbox，否则将RpcMessage封装成OutBoxMessage，主要是工作是序列化RpcMessage，注册回调函数，然后调用NettyRpcEnv.postToOutbox进行数据发送

NettyRpcEnv.ask

```scala
 private[netty] def ask[T: ClassTag](message: RequestMessage, timeout: RpcTimeout): Future[T] = {
    val promise = Promise[Any]()
    val remoteAddr = message.receiver.address

    def onFailure(e: Throwable): Unit = {
      if (!promise.tryFailure(e)) {
        e match {
          case e : RpcEnvStoppedException => logDebug (s"Ignored failure: $e")
          case _ => logWarning(s"Ignored failure: $e")
        }
      }
    }

    def onSuccess(reply: Any): Unit = reply match {
      case RpcFailure(e) => onFailure(e)
      case rpcReply =>
        if (!promise.trySuccess(rpcReply)) {
          logWarning(s"Ignored message: $reply")
        }
    }

    try {
      if (remoteAddr == address) {
        val p = Promise[Any]()
        p.future.onComplete {
          case Success(response) => onSuccess(response)
          case Failure(e) => onFailure(e)
        }(ThreadUtils.sameThread)
        dispatcher.postLocalMessage(message, p)
      } else {
        val rpcMessage = RpcOutboxMessage(message.serialize(this),
          onFailure,
          (client, response) => onSuccess(deserialize[Any](client, response)))
        postToOutbox(message.receiver, rpcMessage)
        promise.future.failed.foreach {
          case _: TimeoutException => rpcMessage.onTimeout()
          case _ =>
        }(ThreadUtils.sameThread)
      }

      val timeoutCancelable = timeoutScheduler.schedule(new Runnable {
        override def run(): Unit = {
          onFailure(new TimeoutException(s"Cannot receive any reply from ${remoteAddr} " +
            s"in ${timeout.duration}"))
        }
      }, timeout.duration.toNanos, TimeUnit.NANOSECONDS)
      promise.future.onComplete { v =>
        timeoutCancelable.cancel(true)
      }(ThreadUtils.sameThread)
    } catch {
      case NonFatal(e) =>
        onFailure(e)
    }
    promise.future.mapTo[T].recover(timeout.addMessageIfTimeout)(ThreadUtils.sameThread)
  }


```

###### 4.2.5 NettyRpcEnv.postToOutbox（目标RpcEndPoint在其他机器）


如果NettyRpcEndpointRef的TransportClient已经初始化，直接通过OutboxMessage.sendWith进行消息的发送, 否则构建Outbox，然后调用Outbox.send进行数据发送


```scala
  private def postToOutbox(receiver: NettyRpcEndpointRef, message: OutboxMessage): Unit = {
    if (receiver.client != null) {
      message.sendWith(receiver.client)
    } else {
      require(receiver.address != null,
        "Cannot send message to client endpoint with no listen address.")
      val targetOutbox = {
        val outbox = outboxes.get(receiver.address)
        if (outbox == null) {
          val newOutbox = new Outbox(this, receiver.address)
          val oldOutbox = outboxes.putIfAbsent(receiver.address, newOutbox)
          if (oldOutbox == null) {
            newOutbox
          } else {
            oldOutbox
          }
        } else {
          outbox
        }
      }
      if (stopped.get) {
        // It's possible that we put `targetOutbox` after stopping. So we need to clean it.
        outboxes.remove(receiver.address)
        targetOutbox.stop()
      } else {
        targetOutbox.send(message)
      }
    }
  }
  
``` 

###### 4.2.5 Outbox.send（目标RpcEndPoint在其他机器）

Outbox.send会将OutboxMessage放入Outbox消息列表，真正的数据发送通过调用Outbox.drainOutbox， 代码如下：

```scala

  def send(message: OutboxMessage): Unit = {
    val dropped = synchronized {
      if (stopped) {
        true
      } else {
        messages.add(message)
        false
      }
    }
    if (dropped) {
      message.onFailure(new SparkException("Message is dropped because Outbox is stopped"))
    } else {
      drainOutbox()
    }
  }
  
```

###### 4.2.6 Outbox.drainOutbox

如果TransportClient没有初始化，会先进行TransportClient进行初始化，然后在调用OutboxMessage.sendWith进行数据发送


```scala

private def drainOutbox(): Unit = {
    var message: OutboxMessage = null
    synchronized {
      if (stopped) {
        return
      }
      if (connectFuture != null) {
        // We are connecting to the remote address, so just exit
        return
      }
      if (client == null) {
        // There is no connect task but client is null, so we need to launch the connect task.
        launchConnectTask()
        return
      }
      if (draining) {
        // There is some thread draining, so just exit
        return
      }
      message = messages.poll()
      if (message == null) {
        return
      }
      draining = true
    }
    while (true) {
      try {
        val _client = synchronized { client }
        if (_client != null) {
          message.sendWith(_client)
        } else {
          assert(stopped == true)
        }
      } catch {
        case NonFatal(e) =>
          handleNetworkFailure(e)
          return
      }
      synchronized {
        if (stopped) {
          return
        }
        message = messages.poll()
        if (message == null) {
          draining = false
          return
        }
      }
    }
  }


```
 



##### 4.3 In方向处理流程(接收数据)

##### 4.3.1 In方向主要链路的代码

```
                  NettyRpcHandler.receive                               
                            |
                            |
                            V   		
      		NettyRpcHandler.internalReceive 
                            |
                            |
                            V   
               Dispatcher.postRemoteMessage   
                            |
                            |
                            V   
                    Dispatcher.postMessage
                            |
                            |
                            V  
                        MessageLoop      
                            |
                            |
                            V   
                        Inbox.process      
```

##### 4.3.2 NettyRpcHandler.receive & NettyRpcHandler.internalReceive

NettyRpcHandler继承了RpcHandler抽象类，是In方向的入口，调用了internalReceive构造了RequestMessage， 然后再调用Dispatcher.postRemoteMessage将Request路由到相应的Inbox

```scala

 def receive(
      client: TransportClient,
      message: ByteBuffer,
      callback: RpcResponseCallback): Unit = {
      //构造RequestMessage
    val messageToDispatch = internalReceive(client, message)
    //调用Dispatcher进行消息路由
    dispatcher.postRemoteMessage(messageToDispatch, callback)
  }


  private def internalReceive(client: TransportClient, message: ByteBuffer): RequestMessage = {
    val addr = client.getChannel().remoteAddress().asInstanceOf[InetSocketAddress]
    assert(addr != null)
    val clientAddr = RpcAddress(addr.getHostString, addr.getPort)
    val requestMessage = RequestMessage(nettyEnv, client, message)
    //发送端没有启动Transport
    if (requestMessage.senderAddress == null) {
      // Create a new message with the socket address of the client as the sender.
      new RequestMessage(clientAddr, requestMessage.receiver, requestMessage.content)
    } else {
      // The remote RpcEnv listens to some port, we should also fire a RemoteProcessConnected for
      // the listening address
      val remoteEnvAddress = requestMessage.senderAddress
      if (remoteAddresses.putIfAbsent(clientAddr, remoteEnvAddress) == null) {
        dispatcher.postToAll(RemoteProcessConnected(remoteEnvAddress))
      }
      requestMessage
    }
  }
```


##### 4.3.3 Dispatcher.postRemoteMessage
构造RpcCallContext&RpcMessage，然后调用Dispatcher.postMessage将消息路由相应的Inbox

```scala
  def postRemoteMessage(message: RequestMessage, callback: RpcResponseCallback): Unit = {
    val rpcCallContext =
      new RemoteNettyRpcCallContext(nettyEnv, callback, message.senderAddress)
    val rpcMessage = RpcMessage(message.senderAddress, message.content, rpcCallContext)
    postMessage(message.receiver.name, rpcMessage, (e) => callback.onFailure(e))
  }
```


##### 4.3.4  Dispatcher.postMessage 


将RpcMessage放入相应的Inbox消息列表，Inbox消息列表MessageLoop线程处理

```scala
private def postMessage(
      endpointName: String,
      message: InboxMessage,
      callbackIfStopped: (Exception) => Unit): Unit = {
    val error = synchronized {
      val data = endpoints.get(endpointName)
      if (stopped) {
        Some(new RpcEnvStoppedException())
      } else if (data == null) {
        Some(new SparkException(s"Could not find $endpointName."))
      } else {
        data.inbox.post(message)
        receivers.offer(data)
        None
      }
    }
    // We don't need to call `onStop` in the `synchronized` block
    error.foreach(callbackIfStopped)
  }

```
  
##### 4.3.5 Inbox.process 

更具消息类型执行相应的处理


```scala
  def process(dispatcher: Dispatcher): Unit = {
    var message: InboxMessage = null
    inbox.synchronized {
      if (!enableConcurrent && numActiveThreads != 0) {
        return
      }
      message = messages.poll()
      if (message != null) {
        numActiveThreads += 1
      } else {
        return
      }
    }
    while (true) {
      safelyCall(endpoint) {
        message match {
          case RpcMessage(_sender, content, context) =>
            try {
              endpoint.receiveAndReply(context).applyOrElse[Any, Unit](content, { msg =>
                throw new SparkException(s"Unsupported message $message from ${_sender}")
              })
            } catch {
              case e: Throwable =>
                context.sendFailure(e)
                // Throw the exception -- this exception will be caught by the safelyCall function.
                // The endpoint's onError function will be called.
                throw e
            }

          case OneWayMessage(_sender, content) =>
            endpoint.receive.applyOrElse[Any, Unit](content, { msg =>
              throw new SparkException(s"Unsupported message $message from ${_sender}")
            })

          case OnStart =>
            endpoint.onStart()
            if (!endpoint.isInstanceOf[ThreadSafeRpcEndpoint]) {
              inbox.synchronized {
                if (!stopped) {
                  enableConcurrent = true
                }
              }
            }

          case OnStop =>
            val activeThreads = inbox.synchronized { inbox.numActiveThreads }
            assert(activeThreads == 1,
              s"There should be only a single active thread but found $activeThreads threads.")
            dispatcher.removeRpcEndpointRef(endpoint)
            endpoint.onStop()
            assert(isEmpty, "OnStop should be the last message")

          case RemoteProcessConnected(remoteAddress) =>
            endpoint.onConnected(remoteAddress)

          case RemoteProcessDisconnected(remoteAddress) =>
            endpoint.onDisconnected(remoteAddress)

          case RemoteProcessConnectionError(cause, remoteAddress) =>
            endpoint.onNetworkError(cause, remoteAddress)
        }
      }

      inbox.synchronized {
        // "enableConcurrent" will be set to false after `onStop` is called, so we should check it
        // every time.
        if (!enableConcurrent && numActiveThreads != 1) {
          // If we are not the only one worker, exit
          numActiveThreads -= 1
          return
        }
        message = messages.poll()
        if (message == null) {
          numActiveThreads -= 1
          return
        }
      }
    }
  }

```  



