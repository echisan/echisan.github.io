---
title: 初悉Netty
date: 2019-04-19 11:21:47
tags: 
  - java
  - nio 
  - netty
---



## Netty介绍



## Netty核心组件

- Channel
- 回调
- Future
- 时间和ChannelHandler

### Channel

Channel是Java NIO的一个基本构造

> 它代表一个到实体(如一个硬件设备、一个文件、一个网络套接字或者一个能够执行一个或者多个不同的I/O操作的程序组件)的开放连接，如读操作和写操作 。

目前，可以把 Channel 看作是传入(入站)或者传出(出站)数据的载体。因此，它可以被打开或者被关闭，连接或者断开连接。

### 回调

有点抽象的描述，用代码解释的话就是如下，当有一个新的连接已经被建立时，`ChannelHandler`的`channelActive()`回调方法将会被调用。

```java
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println(ctx.channel().remoteAddress());
    }
```

### Future

如果使用过java中的Future应该知道，该对象提供的是一个异步操作，通过future.get()可以获取到该结果。

> JDK 预置了 interface java.util.concurrent.Future，但是其所提供的实现，只允许手动检查对应的操作是否已经完成，或者一直阻塞直到它完成。这是非常繁琐的，所以 Netty提供了它自己的实现——ChannelFuture，用于在执行异步操作的时候使用。

### 事件和ChannelHandler

Netty 使用不同的事件来通知我们状态的改变或者是操作的状态。这使得我们能够基于已经
发生的事件来触发适当的动作。这些动作可能是:

- 记录日志
- 数据转换
- 流控制
- 应用程序逻辑

Netty 是一个网络编程框架，所以事件是按照它们与入站或出站数据流的相关性进行分类的。 可能由入站数据或者相关的状态更改而触发的事件包括: 

- 连接已被激活或者连接失活
- 数据读取
- 用户事件
- 错误事件

出站事件是未来将会触发的某个动作的操作结果，这些动作包括:

- 打开或者关闭到远程节点的连接
- 将数据写到或者冲刷到套接字

![image-20190419122546568](/images/201904/channelHandler-chain.png)



#### 选择器、事件和EventLoop

Netty 通过触发事件将 Selector 从应用程序中抽象出来，消除了所有本来将需要手动编写的派发代码。在内部，将会为每个 Channel 分配一个 EventLoop，用以处理所有事件，包括:

- 注册感兴趣的事件
- 将事件派发给ChannelHandler
- 安排进一步的动作

> EventLoop 本身只由一个线程驱动，其处理了一个 Channel 的所有 I/O 事件，并且在该 EventLoop 的整个生命周期内都不会改变。这个简单而强大的设计消除了你可能有的在 ChannelHandler 实现中需要进行同步的任何顾虑，因此，你可以专注于提供正确的逻辑，用 来在有感兴趣的数据要处理的时候执行。 



## 编写Echo服务器和客户端

![image-20190419123740192](/images/201904/echo-client-server.png)

### 编写Echo server

所有Netty服务器都需要以下两部分

- 至少一个ChannelHandler — 该组件实现了服务器对从客户端接收的数据的处理，即它的业务逻辑
- 引导 — 配置服务器的启动代码。至少，它会将服务器绑定到它要监听连接请求的端口上

#### ChannelHandler 和 业务逻辑

> `如果不捕获异常，会发生什么呢`
>
> 每个 Channel 都拥有一个与之相关联的 ChannelPipeline，其持有一个 ChannelHandler 的 实例链。在默认的情况下，ChannelHandler 会把对它的方法的调用转发给链中的下一个 Channel- Handler。因此，如果exceptionCaught()方法没有被该链中的某处实现，那么所接收的异常将会被 传递到 ChannelPipeline 的尾端并被记录。为此，你的应用程序应该提供至少有一个实现了 exceptionCaught()方法的 ChannelHandler。 

#### 一些关键点

- 针对不同类型的事件来调用ChannelHandler
- 应用程序通过实现或扩展ChannelHandler来挂钩到事件的生命周期，并且提供自定义的应用程序逻辑
- 在架构上，ChannelHandler有助于保持业务逻辑与网络处理代码的分离。简化了开发过程，因为代码必须不断地演化已响应不断变化的需求

#### 引导服务器

具体涉及的内容：

- 绑定到服务器将在其上监听并接受传入连接请求的端口（中文版书上就这么写的，看完我都不懂中文了）

  > 其实就是绑定个端口吧，比如像是socket.bind(8080)的样子(?)

- 配置Channel，以将有关的入站消息通知给`EchoServerHandler`实例



### 编写Echo client

> SimpleChannelInboundHandler 与 ChannelInboundHandler 你可能会想:为什么我们在客户端使用的是 SimpleChannelInboundHandler，而不是在 Echo- 
>
> ServerHandler 中所使用的 ChannelInboundHandlerAdapter 呢?这和两个因素的相互作用有 关:业务逻辑如何处理消息以及 Netty 如何管理资源。 
>
> 在客户端，当 channelRead0()方法完成时，你已经有了传入消息，并且已经处理完它了。当该方 法返回时，SimpleChannelInboundHandler 负责释放指向保存该消息的 ByteBuf 的内存引用。 
>
> 在 EchoServerHandler 中，你仍然需要将传入消息回送给发送者，而 write()操作是异步的，直 到 channelRead()方法返回后可能仍然没有完成(如代码清单 2-1 所示)。为此，EchoServerHandler 扩展了 ChannelInboundHandlerAdapter，其在这个时间点上不会释放消息。 
>
> 消息在 EchoServerHandler 的 channelReadComplete()方法中，当 writeAndFlush()方 法被调用时被释放(见代码清单 2-1)。 

#### 引导客户端



