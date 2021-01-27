---
title: 初悉Netty的组件和设计
date: 2019-04-19 18:13:49
tags:
  - netty
  - java
  - nio
---



## Netty

Netty基于Java NIO的**异步**和**事件驱动**的实现，保证了高负载下应用程序性能的最大化和伸缩性。其次，Netty 也包含了一组设计模式，将应用程序逻辑从网络层解耦，简化了开发过程，同时也最大限度地提高了可测试性、模块化以及代码的可重用性。



## Channel、EventLoop和ChannelFuture

Channel,EventLoop,ChannelFuture这三类合起来可以被认为是netty网络抽象的代表

- Channel — Socket
- EventLoop — 控制流、多线程处理、并发
- ChannelFuture — 异步通知

### Channel接口

基本的 I/O 操作(bind()、connect()、read()和 write())依赖于底层网络传输所提供的原语。在基于 Java 的网络编程中，其基本的构造是 class Socket。Netty 的 Channel 接口所提供的 API，大大地降低了直接使用 Socket 类的复杂性。此外，Channel 也是拥有许多预定义的、专门化实现的广泛类层次结构的根，下面是一个简短的部分清单:

- EmbeddedChannel
- LocalServerChannel
- NioDatagramChannel
- NioSctpChannel
- NioSocketChannel

### EventLoop接口

EventLoop 定义了 Netty 的核心抽象，用于处理连接的生命周期中所发生的事件。图 3-1在高层次上说明了 Channel、EventLoop、Thread 以及 EventLoopGroup 之间的关系。

![image-20190419182604710](/images/201904/image-20190419182604710.png)

- 一个 EventLoopGroup 包含一个或者多个 EventLoop;
- 一个 EventLoop 在它的生命周期内只和一个 Thread 绑定;
- 所有由 EventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理;
- 一个 Channel 在它的生命周期内只注册于一个 EventLoop;
- 一个 EventLoop 可能会被分配给一个或多个 Channel。

> 注意，在这种设计中，一个给定 Channel 的 I/O 操作都是由相同的 Thread 执行的，实际 上消除了对于同步的需要。



### ChannelFuture接口

Netty 中所有的 I/O 操作都是异步的。因为一个操作可能不会立即返回，所以我们需要一种用于在之后的某个时间点确定其结果的方法。为此，Netty 提供了ChannelFuture接口，其addListener()方法注册了一个ChannelFutureListener，以便在某个操作完成时(无论是否成功)得到通知。

> 关于 **ChannelFuture** 的更多讨论
>
>  可以将 ChannelFuture 看作是将来要执行的操作的结果的
> 占位符。它究竟什么时候被执行则可能取决于若干的因素，因此不可能准确地预测，但是可以肯
> 定的是它将会被执行。此外，所有属于同一个 Channel 的操作都被保证其将以它们被调用的顺序
> 被执行。



## ChannelHandler and ChannelPipeline

### ChannelHandler

从应用程序开发人员的角度来看，Netty 的主要组件是 ChannelHandler，它充当了所有处理入站和出站数据的应用程序逻辑的容器。这是可行的，因为 ChannelHandler 的方法是由网络事件(其中术语“事件”的使用非常广泛)触发的。事实上，ChannelHandler 可专门用于几乎任何类型的动作，例如将数据从一种格式转换为另外一种格式，或者处理转换过程中所抛出的异常。

举例来说，ChannelInboundHandler 是一个你将会经常实现的子接口。这种类型的ChannelHandler 接收入站事件和数据，这些数据随后将会被你的应用程序的业务逻辑所处理。当你要给连接的客户端发送响应时，也可以从 ChannelInboundHandler 冲刷数据。你的应用程序的业务逻辑通常驻留在一个或者多个ChannelInboundHandler 中。



### ChannelPipeline

ChannelPipeline提供了ChannelHandler链的容器，并定义了用于在该链上传播入站和出站事件流的 API。当 Channel 被创建时，它会被自动地分配到它专属的 ChannelPipeline。

ChannelHandler 安装到 ChannelPipeline 中的过程如下所示:

1. 一个ChannelInitializer的实现被注册到了ServerBootstrap中
2. 当 ChannelInitializer.initChannel()方法被调用时，ChannelInitializer 将在 ChannelPipeline 中安装一组自定义的 ChannelHandler 
3. ChannelInitializer 将它自己从 ChannelPipeline 中移除