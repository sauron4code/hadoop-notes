
--------------------------------------------------------
#### 1 network-common介绍
network-common模块是基于netty实现的节点间通信框架，spark rpc模块基于network-common模块实现，network-common模块是spark的基石


--------------------------------------------------------
#### 2 使用方式
为了让读者有直观的认识，这里写一个测试例子（先从使用开始嘛）
```
环境 
1. IntelliJ IDEA
2. spark 2.4

ps: 为了方便调试，本人在IntelliJ IDEA上打开了两个spark2.4项目, MyServer.java MyClient.java在不同的项目创建、运行、调试，这样就能清晰地调试client，server端的代码
```

##### 2.1 client端代码<span id="myclient"></span>
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


##### 2.2 server端代码<span id="myserver"></span>
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


##### 2.3 测试结果
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

#### 3 network-common源码的目录结构
```shell
buffer： 数据缓冲区(一般用于表示消息的内容体(Message body))
client： client端
protocol：消息协议
server：server端
util: 工具类
```

--------------------------------------------------------


#### 4 buffer

buffer的代码比较简单，涉及ManagedBuffer，FileSegmentManagedBuffer， NettyManagedBuffer， NioManagedBuffer,一般用于表示消息的内容体(Message body)


##### 4.1 ManagedBuffer
ManagedBuffer以字节的形式为数据提供不可变的视图
##### 4.2 ManagedBuffer三种实现：
* 1.FileSegmentManagedBuffer：以文件的形式提供数据
* 2.NioManagedBuffer：以NIO ByteBuffer的形式提供数据
* 3.NettyManagedBuffer：以Netty ByteBuf的形式提供数据 

##### 4.3 uml类图如下图
![avatar](../images/spark/network-common/buffer.png)


##### 4.4 ManagedBuffer源码注释

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

#### 5 protocol
protocol 定义了消息的协议，例如消息类型，消息编码、解码。

##### 5.1 <span id="message_uml">uml类图如下图</span>

![avatar](../images/spark/network-common/protocol.png)

##### 5.2 消息介绍
以上uml类图看起来非常复杂，其实很简单，接下来从消息的分类、编码解码这两方面来介绍

###### 5.2.1 消息分类：
- client端到server端的Request消息：
  - ChunkFetchRequest：向server发送获取流中单个块的请求消息
  - RpcRequest：向server端发送rpc请求消息，由server端的RpcHandler处理
  - StreamRequest：向server端发送获取流式数据的请求消息
  - UploadStream：向server端发送带有数据的Rpc请求消息

- server端到client端的Response消息：
  - Success-Reponse：
	 - ChunkFetchSuccess：处理ChunkFetchRequest成功后返回的响应消息
	 - RpcResponse：处理RpcRequest/UploadStream成功后返回的响应消息
	 - StreamResponse：处理StreamRequest成功后返回的响应消息

  - Fail-Reponse：
  	 - ChunkFetchFailure：处理ChunkFetchRequest失败后返回的响应消息
  	 - RpcFailure：处理RpcRequest/UploadStream失败后返回的响应消息
  	 - StreamFailure：处理StreamRequest失败后返回的响应消息



- 消息类型定义在Message类中的枚举类Type，具体代码如下：

   ```java
	enum Type implements Encodable {
    	ChunkFetchRequest(0), ChunkFetchSuccess(1), ChunkFetchFailure(2),
    	RpcRequest(3), RpcResponse(4), RpcFailure(5),
    	StreamRequest(6), StreamResponse(7), StreamFailure(8),
    	OneWayMessage(9), UploadStream(10), User(-1);

    	private final byte id;

    	Type(int id) {
      		assert id < 128 : "Cannot have more than 128 message types";
     	 	this.id = (byte) id;
    	}
	
    	public byte id() { return id; }
		
		//编码后的字节数	
    	@Override public int encodedLength() { return 1; }
	
		// 将Type对象编码到ByteBuf
    	@Override public void encode(ByteBuf buf) { buf.writeByte(id); }

		// 从ByteBuf解码Type对象
    	public static Type decode(ByteBuf buf) {
      		byte id = buf.readByte();
      		switch (id) {
        		case 0: return ChunkFetchRequest;
        		case 1: return ChunkFetchSuccess;
        		case 2: return ChunkFetchFailure;
        		case 3: return RpcRequest;
        		case 4: return RpcResponse;
        		case 5: return RpcFailure;
        		case 6: return StreamRequest;
        		case 7: return StreamResponse;
        		case 8: return StreamFailure;
        		case 9: return OneWayMessage;
        		case 10: return UploadStream;
        		case -1: throw new IllegalArgumentException("User type messages cannot be decoded.");
        		default: throw new IllegalArgumentException("Unknown message type: " + id);
      		}
    	}
  ```
  
  
###### 5.2.2 消息的定义与消息的编码解码（针对单个消息）： 

- 消息的定义在Message接口中，如下代码：

	```java
	public interface Message extends Encodable {
  			//消息类型（ChunkRequest,StreamRequest等）
  			Type type();

  			//可选的消息体
  			ManagedBuffer body();
				
			//标识消息体跟消息是否在同一帧数据
  			boolean isBodyInFrame();
  			}
	```
		
- 消息的编码解码
	   
	 从[uml类图](#message_uml)图可以看出，Message接口继承了Encodable接口，所有具体消息都必须实现Encodable接口的encodedLength()，encode方法，这就是消息的编码，同时消息也必须提供decode的静态方法，用于消息的解码（MessageDecoder调用）。Encodable的代码如下：
	
	```java
	   public interface Encodable {
  			// 消息编码后的字节数
  			int encodedLength();

   			// 将此消息编码到ByteBuf
  			void encode(ByteBuf buf);
		}
	```


--------------------------------------------------------

#### 6 TransportContext
TransportContext是整个network-common模块的入口类，从[MyClient.java](#myclient)，[MyServer.java](#myserver)可以看出来，TransportClientFactory、TransportServer都是由TransportContext创建，TransportContext除了创建TransportClientFactory、TransportServer，还对Netty channel的pipelines进行设置（client 和 server通过channel进行通信），channel pipelines定义了client端、server端的读写流程

##### 6.1 Netty channel pipelines初始化

步骤如下：

	1.创建TransportChannelHandler, TransportChannelHandler代理了TransportResponseHandler，TransportRequestHandler

	2.设置channel pipelines：注册MessageEncoder， TransportFrameDecoder， MessageDecoder， TransportChannelHandler， IdleStateHandler

	3.返回TransportChannelHandler


##### 6.2 nettty pipeline 执行顺序

```
ChannelOutboundHandler 按照注册的先后顺序逆序执行
ChannelInboundHandler  按照注册的先后顺序顺序执行
```

MessageEncoder， TransportFrameDecoder， MessageDecoder， TransportChannelHandler，IdleStateHandler等类的uml类图如下图：
![avatar](../images/spark/network-common/channel_pipelines.png)

- MessageEncoder是ChannelOutboundHandler，消息编码器，发送数据的时候执行

- TransportFrameDecoder是ChannelInboundHandler，帧解码器(基于tcp/ip的数据传输会有粘包拆包问题，所以需要TransportFrameDecoder将tcp/ip数据流组装成一个完整的帧)，接收数据的时候执行
- MessageDecoder是ChannelInboundHandler，消息解码器，接收数据的时候执行
- IdleStateHandler既是ChannelInboundHandler， 也是ChannelOutboundHandler，心跳检测器
- TransportChannelHandler是ChannelInboundHandler，代理了TransportResponseHandler，TransportRequestHandler，将RequestMessage交给TransportRequestHandler处理，将ResponseMessage交给TransportResponseHandler处理

ChannelOutboundHandler的执行顺序: IdleStateHandler-> MessageEncoder

ChannelInboundHandler的执行顺序：TransportFrameDecoder -> MessageDecoder -> IdleStateHandler -> TransportChannelHandler


>ChannelOutbondHandler，ChannelInboundHandler的执行顺序跟角色无关，不管是client,server都会执行ChannelOutbondHandler，ChannelInboundHandler，因为client,server都需要接收数据(执行ChannelInboundHandler)和发送数据(执行ChannelOutbondHandler)

设置Netty channel pipelines的 主要代码如下：

```java

  public TransportChannelHandler initializePipeline(
      SocketChannel channel,
      RpcHandler channelRpcHandler) {
    try {
    	// 创建TransportChannelHandler
      TransportChannelHandler channelHandler = createChannelHandler(channel, channelRpcHandler);
      channel.pipeline()
      	 //注册MessageEncoder
        .addLast("encoder", ENCODER)
        //注册TransportFrameDecoder
        .addLast(TransportFrameDecoder.HANDLER_NAME, NettyUtils.createFrameDecoder())
        // 注册MessageDecoder
        .addLast("decoder", DECODER)
        //注册IdleStateHandler
        .addLast("idleStateHandler", new IdleStateHandler(0, 0, conf.connectionTimeoutMs() / 1000))
        //注册TransportChannelHandler
        .addLast("handler", channelHandler);
      return channelHandler;
    } catch (RuntimeException e) {
      logger.error("Error while initializing Netty pipeline", e);
      throw e;
    }
  }


  private TransportChannelHandler createChannelHandler(Channel channel, RpcHandler rpcHandler) {
    TransportResponseHandler responseHandler = new TransportResponseHandler(channel);
    TransportClient client = new TransportClient(channel, responseHandler);
    TransportRequestHandler requestHandler = new TransportRequestHandler(channel, client,
      rpcHandler, conf.maxChunksBeingTransferred());
    return new TransportChannelHandler(client, responseHandler, requestHandler,
      conf.connectionTimeoutMs(), closeIdleConnections);
  }

```


##### 6.3 编码器（发送数据，out方向）
发送数据主要涉及MessageEncoder， MessageEncoder将Message编码成一帧，帧的格式&代码如下：

```
+---------------------------------+-----------------------------------+
|            header               |           body                    |
+---------------------------------+-----------------------------------+
| FrameLength | MsgType | Message |      MessageBody                  |    
+---------------------------------+-----------------------------------+
```

```java
public final class MessageEncoder extends MessageToMessageEncoder<Message> {

  private static final Logger logger = LoggerFactory.getLogger(MessageEncoder.class);

  public static final MessageEncoder INSTANCE = new MessageEncoder();

  private MessageEncoder() {}


  @Override
  public void encode(ChannelHandlerContext ctx, Message in, List<Object> out) throws Exception {
    Object body = null;
    long bodyLength = 0;
    boolean isBodyInFrame = false;

    // If the message has a body, take it out to enable zero-copy transfer for the payload.
    if (in.body() != null) {
      try {
        bodyLength = in.body().size();
        body = in.body().convertToNetty();
        isBodyInFrame = in.isBodyInFrame();
      } catch (Exception e) {
        in.body().release();
        if (in instanceof AbstractResponseMessage) {
          AbstractResponseMessage resp = (AbstractResponseMessage) in;
          // Re-encode this message as a failure response.
          String error = e.getMessage() != null ? e.getMessage() : "null";
          logger.error(String.format("Error processing %s for client %s",
            in, ctx.channel().remoteAddress()), e);
          encode(ctx, resp.createFailureResponse(error), out);
        } else {
          throw e;
        }
        return;
      }
    }
	
	 //消息类型
    Message.Type msgType = in.type();
    //计算header长度
    int headerLength = 8 + msgType.encodedLength() + in.encodedLength();
    //计算frame长度
    long frameLength = headerLength + (isBodyInFrame ? bodyLength : 0);
    //创建header(ByteBuf)
    ByteBuf header = ctx.alloc().heapBuffer(headerLength);
    //将frame长度，MessageType，Message编码到header(ByteBuf)
    header.writeLong(frameLength);
    msgType.encode(header);
    in.encode(header);
    assert header.writableBytes() == 0;

    if (body != null) {
      out.add(new MessageWithHeader(in.body(), header, body, bodyLength));
    } else {
      out.add(header);
    }
  }

}


```

##### 6.4 解码器（接收数据，in方向）

接收数据主要涉及以下流程：
TransportFrameDecoder -> MessageDecoder -> IdleStateHandler -> TransportChannelHandler

TransportFrameDecoder：将tcp/ip数据流组装成一个帧， 因为MessageEncoder将消息编码成一帧才发送，数据接收端由于tcp/ip粘包拆包问题，所以接受端也需要将接收到tcp/ip数据流组装成一帧数据，然后将这帧数据交给MessageDecoder处理，具体代码如下：

```scala
public class TransportFrameDecoder extends ChannelInboundHandlerAdapter {

  public static final String HANDLER_NAME = "frameDecoder";
  private static final int LENGTH_SIZE = 8;
  private static final int MAX_FRAME_SIZE = Integer.MAX_VALUE;
  private static final int UNKNOWN_FRAME_SIZE = -1;

  private final LinkedList<ByteBuf> buffers = new LinkedList<>();
  private final ByteBuf frameLenBuf = Unpooled.buffer(LENGTH_SIZE, LENGTH_SIZE);

  private long totalSize = 0;
  private long nextFrameSize = UNKNOWN_FRAME_SIZE;
  private volatile Interceptor interceptor;

  @Override
  public void channelRead(ChannelHandlerContext ctx, Object data) throws Exception {
    ByteBuf in = (ByteBuf) data;
    buffers.add(in);
    totalSize += in.readableBytes();

    while (!buffers.isEmpty()) {
    	//如果Interceptor已经安装，使用Interceptor处理数据，用于处理StreamResponse/UploadStream
      if (interceptor != null) {
        ByteBuf first = buffers.getFirst();
        int available = first.readableBytes();
        if (feedInterceptor(first)) {
          assert !first.isReadable() : "Interceptor still active but buffer has data.";
        }

        int read = available - first.readableBytes();
        if (read == available) {
          buffers.removeFirst().release();
        }
        totalSize -= read;
      } else {
		 //对tcp/ip数据流进行解码，组装成一帧数据
        ByteBuf frame = decodeNext();
        if (frame == null) {
          break;
        }
        ctx.fireChannelRead(frame);
      }
    }
  }

  private long decodeFrameSize() {
    // 如果nextFrameSize等于UNKNOWN_FRAME_SIZE（-1：说明上一帧还未完成帧解码） 或者 (ByteBuf) data长度小于LENGTH_SIZE（8字节， FrameLength）， 返回nextFrameSize
    if (nextFrameSize != UNKNOWN_FRAME_SIZE || totalSize < LENGTH_SIZE) {
      return nextFrameSize;
    }


    ByteBuf first = buffers.getFirst();
    //如果ByteBuf可读字节数大于等于LENGTH_SIZE（8）， 直接读取FrameLength，返回FrameLength
    if (first.readableBytes() >= LENGTH_SIZE) {
      nextFrameSize = first.readLong() - LENGTH_SIZE;
      totalSize -= LENGTH_SIZE;
      if (!first.isReadable()) {
        buffers.removeFirst().release();
      }
      return nextFrameSize;
    }

    // 如果如果ByteBuf可读字节数小于LENGTH_SIZE（8），将数据缓存到frameLenBuf(ByteBuf)， 当frameLenBuf(ByteBuf)等于LENGTH_SIZE（8）跳出循环
    while (frameLenBuf.readableBytes() < LENGTH_SIZE) {
      ByteBuf next = buffers.getFirst();
      int toRead = Math.min(next.readableBytes(), LENGTH_SIZE - frameLenBuf.readableBytes());
      frameLenBuf.writeBytes(next, toRead);
      if (!next.isReadable()) {
        buffers.removeFirst().release();
      }
    }
    
    // 读取FrameLength，返回FrameLength
    nextFrameSize = frameLenBuf.readLong() - LENGTH_SIZE;
    totalSize -= LENGTH_SIZE;
    frameLenBuf.clear();
    return nextFrameSize;
  }

  private ByteBuf decodeNext() {
    long frameSize = decodeFrameSize();
    if (frameSize == UNKNOWN_FRAME_SIZE || totalSize < frameSize) {
      return null;
    }

    nextFrameSize = UNKNOWN_FRAME_SIZE;


    //如果第一个ByteBuf包含整个帧，直接返回整个帧
    int remaining = (int) frameSize;
    if (buffers.getFirst().readableBytes() >= remaining) {
      return nextBufferForFrame(remaining);
    }

    // 如果帧在多个ByteBuf中，将多个ByteBuf数据缓存到CompositeByteBuf，直到解析到一个完整的帧
    CompositeByteBuf frame = buffers.getFirst().alloc().compositeBuffer(Integer.MAX_VALUE);
    while (remaining > 0) {
      ByteBuf next = nextBufferForFrame(remaining);
      remaining -= next.readableBytes();
      frame.addComponent(next).writerIndex(frame.writerIndex() + next.readableBytes());
    }
    assert remaining == 0;
    return frame;
  }

  private ByteBuf nextBufferForFrame(int bytesToRead) {
    ByteBuf buf = buffers.getFirst();
    ByteBuf frame;

    if (buf.readableBytes() > bytesToRead) {
      frame = buf.retain().readSlice(bytesToRead);
      totalSize -= bytesToRead;
    } else {
      frame = buf;
      buffers.removeFirst();
      totalSize -= frame.readableBytes();
    }

    return frame;
  }


  private boolean feedInterceptor(ByteBuf buf) throws Exception {
    if (interceptor != null && !interceptor.handle(buf)) {
      interceptor = null;
    }
    return interceptor != null;
  }
}

```



MessageDecoder：从TransportFrameDecoder接收数据，解码成Message，然后交给TransportChannelHandler处理，代码如下：

```scala
public final class MessageDecoder extends MessageToMessageDecoder<ByteBuf> {

  private static final Logger logger = LoggerFactory.getLogger(MessageDecoder.class);

  public static final MessageDecoder INSTANCE = new MessageDecoder();

  private MessageDecoder() {}

  @Override
  public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    //解码MessageType
    Message.Type msgType = Message.Type.decode(in);
    //解码Message
    Message decoded = decode(msgType, in);
    assert decoded.type() == msgType;
    logger.trace("Received message {}: {}", msgType, decoded);
    out.add(decoded);
  }

  private Message decode(Message.Type msgType, ByteBuf in) {
    switch (msgType) {
      case ChunkFetchRequest:
        return ChunkFetchRequest.decode(in);

      case ChunkFetchSuccess:
        return ChunkFetchSuccess.decode(in);

      case ChunkFetchFailure:
        return ChunkFetchFailure.decode(in);

      case RpcRequest:
        return RpcRequest.decode(in);

      case RpcResponse:
        return RpcResponse.decode(in);

      case RpcFailure:
        return RpcFailure.decode(in);

      case OneWayMessage:
        return OneWayMessage.decode(in);

      case StreamRequest:
        return StreamRequest.decode(in);

      case StreamResponse:
        return StreamResponse.decode(in);

      case StreamFailure:
        return StreamFailure.decode(in);

      case UploadStream:
        return UploadStream.decode(in);

      default:
        throw new IllegalArgumentException("Unexpected message type: " + msgType);
    }
  }
}
```

TransportChannelHandler：消息（RpcRequest，ChunkRequest等消息）的处理器，主要代码如下：

```
public class TransportChannelHandler extends ChannelInboundHandlerAdapter {

  @Override
  public void channelRead(ChannelHandlerContext ctx, Object request) throws Exception {
  	 //如果是RequestMessage， 交给RequestHandler处理
    if (request instanceof RequestMessage) {
      requestHandler.handle((RequestMessage) request);
    } else if (request instanceof ResponseMessage) {  	 //如果是RequestMessage， 交给ResponseHandler处理
      responseHandler.handle((ResponseMessage) request);
    } else {
      ctx.fireChannelRead(request);
    }
  }
  
}  
```

至此关于消息的读写流程已经分析得七七八八，接下来将分析Rpc，Stream，Chunk的完整读写流程

--------------------------------------------------------

#### 7 Rpc请求的完整流程

```

   client端
+-------------------------+
| TransportClient.sendRpc |
+-------------------------+
  |                                   
  |                                   
  v                                   
+-------------------------+
|    IdleStateHandler     |
+-------------------------+
  |                                   
  |                                   
  v                                   
+-------------------------+
|  MessageEncoder.encode  |
+-------------------------+
   |
   |                                                                           server端
   |                      channel               +--------------------------------------------------+
   |------------------------------------------->|        TransportFrameDecoder.channelRead         |
                                                +--------------------------------------------------+
                                                  |
                                                  |
                                                  v
                                                +--------------------------------------------------+
                                                |              MessageDecoder.decode               |
                                                +--------------------------------------------------+
                                                  |
                                                  |
                                                  v
                                                +--------------------------------------------------+
                                                |       TransportChannelHandler.channelRead        |
                                                +--------------------------------------------------+
                                                  |
                                                  |
                                                  v
                                                +--------------------------------------------------+
                                                | TransportRequestHandler.handle.processRpcRequest |
                                                +--------------------------------------------------+
                                                  |
                                                  |
                                                  v
                                                +--------------------------------------------------+
                                                |                RpcHandler.receive                |
                                                +--------------------------------------------------+
                                                  |
                                                  |
                                                  v
                                                +--------------------------------------------------+
                                                |                 IdleStateHandle                  |
                                                +--------------------------------------------------+
                                                  |
                                                  |
                                                  v
                       channel                  +--------------------------------------------------+
     -------------------------------------------|              MessageEncoder.encode               |
     |                                          +--------------------------------------------------+
     |                                                                                           
     |
     v  client端
+-------------------------------------+                              
|  TransportFrameDecoder.channelRead  |                              
+-------------------------------------+                              
  |                                                                   
  |                                                                    
  v                                                                   
+-------------------------------------+                              
|        MessageDecoder.decode        |                              
+-------------------------------------+                              
  |                                                                  
  |                                                                  
  v                                                                   
+-------------------------------------+                              
| TransportChannelHandler.channelRead |                              
+-------------------------------------+                              
  |                                                                  
  |                                                                  
  v                                                                  
+-------------------------------------+                              
|   TransportResponseHandler.handle   |                              
+-------------------------------------+                              
  |                                                                  
  |                                                                  
  v                                                                  
+-------------------------------------+                              
|    RpcResponseCallback.onSuccess    |                              
+-------------------------------------+                              
						
```
以上是一个Rpc请求的完整流程，Chunk、Stream请求流程都类似，这里就不一一展开了。
















 

