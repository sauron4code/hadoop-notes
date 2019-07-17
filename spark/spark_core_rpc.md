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