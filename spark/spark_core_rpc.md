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
```

##### 4.2.2 RpcEnv.setupEndpointRef 

RpcEnv.setupEndpointRef 主要的作用是创建EndpointRef，并检查目的Endpoint是否存在，这个通过RpcEndpointVerifier， 每个NettyRpcEnv在启动TransportServer时都会注册RpcEndpointVerifier


###### 4.2.2.1 RpcEnv.setupEndpointRef（ 注册RpcEndpointVerifier ）代码如下：

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

###### 4.2.2.3 NettyRpcEndpointRef.ask（向目标Endpoint发送Rpc请求）



##### 4.3 In方向处(接收数据)理流程
