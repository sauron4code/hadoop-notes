
--------------------------------------------------------
#### 1.network-common介绍
network-common模块是基于netty实现的节点间通讯框架，spark rpc模块基于network-common模块实现，network-common模块是spark的基石。


--------------------------------------------------------
#### 2.使用方式
为了让读者有个直观的认识，这里写一个测试例子（先从使用开始嘛）
```
环境 
1. IntelliJ IDEA
2. spark 2.4

ps: 为了方便调试，本人在IntelliJ IDEA上打开了两个spark2.4项目, MyServer.java MyClient.java在不同的项目创建、运行、调试
```

#### 2.1 client端代码
```scala
/**
client使用流程：
1.创建TransportConf 和 TransportContext
2.通过TransportContext创建TransportClientFactory
3.通过TransportClientFactory创建TransportClient
4.通过TransportClient注册Callback和发送请求（Stream请求、Chunk请求、Rpc请求）
5.Server端返回，执行Callback




**/

package org.apache.spark.network.mytest;

import org.apache.spark.network.TransportContext;
import org.apache.spark.network.buffer.ManagedBuffer;
import org.apache.spark.network.client.*;
import org.apache.spark.network.server.RpcHandler;
import org.apache.spark.network.server.StreamManager;
import org.apache.spark.network.util.ConfigProvider;
import org.apache.spark.network.util.JavaUtils;
import org.apache.spark.network.util.MapConfigProvider;
import org.apache.spark.network.util.TransportConf;
import org.junit.Test;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.util.HashMap;

public class MyClient {

    @Test
    public void client() {

        // 创建空的TransportConf
        ConfigProvider configProvider = new MapConfigProvider(new HashMap<String, String>());
        TransportConf conf = new TransportConf("test", configProvider);


        // 创建RpcHandler，用于创建TransportContext，此测试用例的client端没有用到RpcHandler，所以这个RpcHandler为空
        RpcHandler rpcHandler = new RpcHandler() {
            @Override
            public void receive(TransportClient client, ByteBuffer message, RpcResponseCallback callback) {

            }

            @Override
            public StreamManager getStreamManager() {
                return null;
            }
        };

        // 创建TransportContext
        TransportContext transportContext = new TransportContext(conf, rpcHandler);

        // 创建TransportClientFactory
        TransportClientFactory transportClientFactory = transportContext.createClientFactory();

        try {

            // 创建连接到127.0.0.1:5633-server端的client
            TransportClient transportClient =
                    transportClientFactory.createClient("127.0.0.1", 5633);

            // 向server端发送内容为"RpcRpc"的Rpc请求，设置回调函数，回调函数将sever端返回的消息打印在控制台
            transportClient.sendRpc(JavaUtils.stringToBytes("RpcRpc"), new RpcResponseCallback() {
                @Override
                public void onSuccess(ByteBuffer response) {
                    String result = JavaUtils.bytesToString(response);
                    System.out.println("rpc response:  " + result);
                }

                @Override
                public void onFailure(Throwable e) {
                    e.printStackTrace();

                }
            });



            // 向server端发送StreamId=1，ChunkIndex=1(这里的StreamId,ChunkIdex可以随便设置，因为测试Server端只会返回内容为"ChunkChunk"的Chunk响应)的Chunk请求，
            // 设置回调函数，回调函数将sever端返回的消息打印在控制台
            transportClient.fetchChunk(1, 1, new ChunkReceivedCallback() {
                @Override
                public void onSuccess(int chunkIndex, ManagedBuffer buffer) {
                    try {
                        System.out.println("chunk response: " + JavaUtils.bytesToString(buffer.nioByteBuffer()));
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }

                @Override
                public void onFailure(int chunkIndex, Throwable e) {

                }
            });

            // 向server端发送StreamId="1"(这里的StreamId可以随便设置，因为测试Server端只会返回内容为"StreamStream"的Stream响应)的Stream请求，
            // 设置回调函数，回调函数将sever端返回的消息打印在控制台
            transportClient.stream("1", new StreamCallback() {
                @Override
                public void onData(String streamId, ByteBuffer buf) throws IOException {
                    System.out.println("stream response: " + JavaUtils.bytesToString(buf));

                }

                @Override
                public void onComplete(String streamId) throws IOException {
                }

                @Override
                public void onFailure(String streamId, Throwable cause) throws IOException {
                    cause.printStackTrace();

                }
            });


        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }


        // 主线程睡眠5s，等待netty线程结束
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}


```


#### 2.2 server端代码
```scala
/**
server使用流程：

1.创建TransportConf
2.创建StreamManager，用于处理Stream/Chunk请求
3.创建RpcHandler，用于处理Rpc请求
4.创建TransportContext
5.通过TransportContext创建TransportServer



**/
package org.apache.spark.network.mytest;

import com.sun.corba.se.internal.CosNaming.BootstrapServer;
import io.netty.channel.Channel;
import org.apache.spark.network.TransportContext;
import org.apache.spark.network.buffer.ManagedBuffer;
import org.apache.spark.network.buffer.NioManagedBuffer;
import org.apache.spark.network.client.RpcResponseCallback;
import org.apache.spark.network.client.TransportClient;
import org.apache.spark.network.server.RpcHandler;
import org.apache.spark.network.server.StreamManager;
import org.apache.spark.network.server.TransportServer;
import org.apache.spark.network.server.TransportServerBootstrap;
import org.apache.spark.network.util.ConfigProvider;
import org.apache.spark.network.util.JavaUtils;
import org.apache.spark.network.util.MapConfigProvider;
import org.apache.spark.network.util.TransportConf;
import org.junit.Test;

import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashMap;

public class MyServer {

    @Test
    public void servers() {

        // 创建空的TransportConf
        ConfigProvider configProvider = new MapConfigProvider(new HashMap<String, String>());
        TransportConf conf = new TransportConf("test", configProvider);


        // 创建StreamMangere，用于响应Stream/Chunk请求
        StreamManager streamManager = new StreamManager() {
            @Override
            public ManagedBuffer getChunk(long streamId, int chunkIndex) {
                return new NioManagedBuffer(JavaUtils.stringToBytes("ChunkChunk"));
            }

            @Override
            public ManagedBuffer openStream(String streamId) {
                return new NioManagedBuffer(JavaUtils.stringToBytes("StreamStream"));
            }
        };


        // 创建RpcHandler，用于响应，设置回调函数函数，回调函数将client端Rpc请求的消息原样返回
        RpcHandler rpcHandler = new RpcHandler() {
            @Override
            public void receive(TransportClient client, ByteBuffer message, RpcResponseCallback callback) {
                String requestBody = JavaUtils.bytesToString(message);
                callback.onSuccess(JavaUtils.stringToBytes( requestBody));
            }

            @Override
            public StreamManager getStreamManager() {
                return streamManager;
            }
        };

        // 创建TransportContext
        TransportContext  transportContext = new TransportContext(conf, rpcHandler);

        // 创建在127.0.0.1:5633监听的server

        TransportServer transportServer = transportContext.createServer("127.0.0.1", 5633, new ArrayList<TransportServerBootstrap>() );

        // 主线程睡眠10s
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}

```


#### 2.3 测试结果
- client端的输出
```
rpc response:  RpcRpc
chunk response: ChunkChunk
stream response: StreamStream

Process finished with exit code 0

```

- server端的输出
```
Process finished with exit code 0
```
--------------------------------------------------------

#### 3. network-common源码的目录结构
```shell
buffer： 数据缓冲区(一般用于表示消息的内容体(Message body))
client： client端
protocol：消息协议
server：server端
util: 工具类
```


##### 3.1 buffer

buffer的代码比较简单，涉及ManagedBuffer，FileSegmentManagedBuffer， NettyManagedBuffer， NioManagedBuffer,一般用于表示消息的内容体(Message body)


##### 3.1.1 ManagedBuffer
ManagedBuffer以字节的形式为数据提供不可变的视图
##### 3.1.2 ManagedBuffer三种实现：
* 1.FileSegmentManagedBuffer：以文件的形式提供数据
* 2.NioManagedBuffer：以NIO ByteBuffer的形式提供数据
* 3.NettyManagedBuffer：以Netty ByteBuf的形式提供数据 

#### 3.1.3 uml如下图
![avatar](../images/spark/network-common/buffer.png)


#### 3.1.4 ManagedBuffer源码注释

```java
public abstract class ManagedBuffer {
  // ManagedBuffer的字节数
  public abstract long size();

  // 将ManagedBuffer的数据暴露为NIO ByteBuffer
  public abstract ByteBuffer nioByteBuffer() throws IOException;

  // 将ManagedBuffer的数据暴露为InputStream
  public abstract InputStream createInputStream() throws IOException;


  // 增加ManagedBuffer的引用计数
  public abstract ManagedBuffer retain();

  // 减少ManagedBuffer的引用计数
  public abstract ManagedBuffer release();

  //将ManagedBuffer转换为Netty对象，返回值为ByteBuf或者FileRegion
  public abstract Object convertToNetty() throws IOException;
}


```
--------------------------------------------------------



 

