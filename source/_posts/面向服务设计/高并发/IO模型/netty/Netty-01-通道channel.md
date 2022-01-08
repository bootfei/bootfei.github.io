---
title: Netty-01-通道channel
date: 2021-05-10 21:42:41
tags:
---



# 通道Channel概述

------

我们从如下几个方面来简单了解一下 Channel。

- 通道的当前状态：open(端口打开)、connect(连接)。
- 通道的配置：包含通道的配置属性与网络通信选项(ChannelOption)。
- IO 通道方法：read、write、connect、bind 与管道(ChannelPipeline)等。
- 所有 IO 操作在 Netty 中都是异步的：调用 IO 方法例如 write 方法后，并不是等 IO 操作实际完成后再返回，而是会立即返回一个凭证，IO 操作完成后会将结果写入凭证中，典型的 Future设计模式。
- Channel 具有父子关系：所有的 SocketChannel（客户端发起TCP连接）都是由 ServerSocketChannel（服务端接收连接）接收客户端连接而创建的，SocketChannel 的 parent() 方法会返回对应的 ServerSocketChannel。
- 所有通道对象在使用完后，请务必调用通道的colse方法来释放资源。

本节将从如下3个方面来重点介绍Channel。

- Channel 常用API
- Channel 配置与选项
- NIO相关的Channel继承图

## Channel常用API

核心API一览：

- EventLoop eventLoop()
  返回该通道注册的事件轮询器。
- Channel parent()
  返回该通道的父通道，如果是ServerSocketChannel实例则返回null，SocketChannel实例则返回对应的ServerSocketChannel。
- ChannelConfig config()
  返回该通道的配置参数。
- boolean isOpen()
  端口是否处于open，通道默认一创建isOpen方法就会返回true，close方法被调用后该方法返回false。
- boolean isRegistered()
  是否已注册到EventLoop。
- public boolean isActive()
  通道是否处于激活。[NioSocketChannel的实现是java.nio.channels.SocketChannel]()，实例的isOpen()与isConnected()都返回true。NioServerSocketChannel的实现是ServerSocketChannel.socket().isBound()，如果绑定到端口中，意味着处于激活状态。
- ChannelFuture closeFuture()
  Future 模式的应用，调用该方法的目的并不是关闭通道，而是预先创建一个凭证(Future)，等通道关闭时，会通知该 Future，用户可以通过该 Future 注册事件。
- ChannelFuture bind(SocketAddress localAddress)
  Netty 服务端绑定到本地端口，开始监听客户端的连接请求。该过程会触发事件链(ChannelPipeline)。该部分将在后续讲解服务端启动流程时再详细分析。
- ChannelFuture connect(SocketAddress remoteAddress)
  Netty客户端连接到服务端，该过程同样会触发一系列事件(ChannelPipeline)。该部分将在后续讲解客户端启动流程时再详细分析。
- ChannelFuture disconnect()
  断开连接，但不会释放资源，该通道还可以再通过connect重新与服务器建立连接。
- ChannelFuture close()
  关闭通道，回收资源，该通道的生命周期完全结束。
- ChannelFuture deregister()
  取消注册。
- Channel read()
  通道读，该方法并不是直接从读写缓存区读取文件，而是向NIO Selecor注册读事件（目前主要基于NIO）。当通道收到对端的数后，事件选择器会处理读事件，从而触发ChannelInboundHandler#channelRead 事件，然后继续触发ChannelInboundHandler#channelReadComplete(ChannelHandlerContext)事件。
- ChannelFuture write(Object msg)
  向通道写字节流，会触发响应的写事件链，该方法只是会将字节流写入到通道缓存区，并不会调用flush方法写入通道中。
- Channel flush()
  刷写所有挂起的消息（刷写到流中）。
- ChannelFuture writeAndFlush(Object msg)
  相当于调用write与flush方法。

## Channel配置与选项

#### Channel配置

核心配置如下：

- Map<ChannelOption<?>, Object> options：网络相关的配置属性。

  <channeloption</channeloption

- int connectTimeoutMillis：连接超时时间。

- int maxMessagesPerRead：每次读事件中调用读方法的最大次数(AbstractNioByteChannel)或读事件循环中最多处理的消息条数(AbstractNioMessageChannel)。

- int writeSpinCount：一次写事件处理期间最多调用write方法的次数，引入该机制主要是为了避免一个网络通道写入大量数据，对其他网络通道的读写处理带来延迟，默认值为16。

- ByteBufAllocator getAllocator()：返回该通道的内存分配器(ByteBuf)。
  RecvByteBufAllocator getRecvByteBufAllocator()：读事件读缓冲区的分配策略。

- boolean autoRead：是否自动触发read方法调用，默认为true，读事件触发后自动调用read方法 ，而无需应用程序显示调用。

- int writeBufferHighWaterMark：设置写缓存区的高水位线。如果写缓存区中的数据超过该值，Channel#isWritable()方法将返回false。

- int writeBufferLowWaterMark：设置写缓存区的低水位线。如果写缓存区的数据超过高水位线后，通道将变得不可写，等写缓存数据降低到低水位线后通道恢复可写状态(Channel#isWritable()将再次返回true)。

#### ChannelOption

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)网络通道(Channel)选项值，下面介绍一下与TCP协议相关的核心参数：

- SO_BROADCAST
  选择值类型:boolean。表示该数据包是否是广播包，true表示广播包，false表示非广播，如果包的IP地址为广播地址，但该选型为false，则在内核层会抛出错误。

- SO_KEEPALIVE 对于面向连接的TCP socket,在实际应用中通常都要检测对端是否处于连接中,连接端口分两种情况:

- - 连接正常关闭,调用close() shutdown()连接优雅关闭,send与recv立马返回错误,select返回SOCK_ERR
  - 连接的对端异常关闭,比如网络断掉,突然断电.

- SO_SNDBUF的大小
  为了达到最大网络吞吐，socket send buffer size(SO_SNDBUF)不应该小于带宽和延迟的乘积。

- SO_REUSEADDR
  该参数如果设置为true的一个常用应用场景是端口复用(直接复用TIME_WAIT状态的socket)。

- SO_LINGER
  该参数是控制TCP关闭行为的。

- SO_BACKLOG
  服务端接受客户端连接的处理队列，在TCP三次握手协议中，服务端接收到客户端的SYN包后，会向客户端发送SYN+ACK包，同时会将连接放入到 backlog 队列中，等待客户端ACK包。在服务端没有接收到客户端的ACK包之前，连接会暂存 backlog 队列。

- SO_TIMEOUT
  以毫秒为单位定义套接字超时(SO_TIMEOUT)，它是等待数据的超时，或者换句话说，是两个连续数据包之间的最大活动周期。超时值为0将被解释为无限超时。如果没有设置该参数，读取操作将不会超时(无穷小超时)。个人思考：在NIO编程开发中应该不要设置该值，但为了保证每个连接的读平等，Netty会控制一次事件选择周期，最多可调用read方法的次数。

- TCP_NODELAY
  在 TCP 数据包发送的时候，有一种算法（Nagle算法）。该算法的核心是如果发生数据包比较小，为了提高带宽的利用率，会等待更多的数据到达后再发送或等待超时后将小包发送，也就是 TCP 发送延迟，TCP_NODELAY = true表示不使用 tcp delay 延迟，故禁用 Nagle 算法。通常接受端的 ACK 包也会使用延迟（默认40ms)，旨在合并多个 ACK 确认包。

## Channel NIO 继承图

------

Channel 类继承图主要是想展示一下与 NIO 相关的 NioSocketChannel (客户端通道)与NioServerSocketChannel (服务端通道)在 Channel 中的位置。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFt0gMRjoLu0IMsRO5PUVDRcLBlmu62NEjNZMrARJCsMNf7kibDjeq9EWYIz6wYBezwaF3tPBfkS0Ug/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Channel 通道统一抽象接口。

- AbstractChannel 通道默认抽象实现类
- AbstractEpollChannel unix Epoll通道实现
- AbstractOioChannel 阻塞IO通道抽象类
- AbstractNioChannel NIO通道抽象类
- AbstractNioByteChannel NIO客户端通道抽象类
- AbstractNIoMessageChannel NIO服务端通道抽象类
- NioSocketChannel  NIO客户端通道实现类
- NioServerSocketChannel NIO服务端通道实现类





# ChannelHandler概述

本节主要介绍 Netty ChannelHandler 事件概述，并详细介绍各个事件方法的触发时机，为下篇关于事件传播机制打下坚实基础。

NIO 相关的核心类图如下：

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/Wkp2azia4QFtZyzkSTqa2JIFiaMd1TMNpRwY0FL4nqaqhZBDicScXGmKSe5d99mdJbJqHO3picibBb9sz3BBetnTqibQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## ChannelHandler

- ChannelHandler：Netty Channel 事件的基础接口，只定义与 Handler 的管理接口相关，具体如下：

- - void handlerAdded(ChannelHandlerContext ctx)
    在调用 DefaultChannelPipeline 的 addLast(add*) 将事件监听器添加到事件处理链条时调用。
  - void handlerRemoved(ChannelHandlerContext ctx)
    在调用DefaultChannelPipeline 的 addLast(add*) 发生异常时被调用；当通道关闭后，通道取消注册后，同时会触发通道移除事件，具体调用入口：DefaultChannelPipeline 的内部类 HeadContext 的 channelUnregistered。

## ChannelIn(Out)boundHandler

- ChannelInboundHandler(入端类型的事件处理器)

- - void channelRegistered(ChannelHandlerContext ctx) 
    通道注册到 Selector 时触发。客户端在调用 connect 方法，通过 TCP 建立连接后，获取 SocketChannel 后将该通道注册在 Selector 时或服务端在调用bind 方法后创建 ServerSocketChannel，通过将通道注册到 Selector 时监听客户端连接上时被调用。
  - void channelUnregistered(ChannelHandlerContext ctx)
    通道取消注册到Selector时被调用，通常在通道关闭时触发，首先触发channelInactive 事件，然后再触发 channelUnregistered 事件。
  - void channelActive(ChannelHandlerContext ctx)
    通道处于激活的事件，在 Netty 中，处于激活状态表示底层 Socket 的isOpen() 方法与 isConnected() 方法返回 true。
  - void channelInactive(ChannelHandlerContext ctx)
    通道处于非激活（关闭），调用了 close 方法时，会触发该事件，然后触发channelUnregistered 事件。
  - void channelRead(ChannelHandlerContext ctx, Object msg)
    通道从对端读取数据，当事件轮询到读事件，调用底层 SocketChanne 的 read 方法后，将读取的字节通过事件链进行处理，NIO 的触发入口为AbstractNioByteChannel 的内部类 NioByteUnsafe 的 read 方法。
  - void channelReadComplete(ChannelHandlerContext ctx)
    处理完一次通道读事件后触发，在 Netty 中一次读事件处理中，会多次调用SocketChannel 的 read方法。触发入口为AbstractNioByteChannel 的内部类NioByteUnsafe 的 read 方法。
  - void userEventTriggered(ChannelHandlerContext ctx, Object evt) 
    触发用户自定义的事件，目前只定义了ChannelInputShutdownEvent（如果允许半关闭（输入端关闭而服务端不关闭））事件。
  - void channelWritabilityChanged(ChannelHandlerContext ctx)
    Netty 写缓存区可写状态变更事件（可写--》不可写、不可写--》可写），入口消息发送缓存区ChannelOutboundBuffer。
  - void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) 
    异常事件。

- ChannelOutboundHandler出端类型的事件处理器。

- - void bind(ChannelHandlerContext ctx, SocketAddress add, ChannelPromise p)
    调用ServerBootstrap 的 bind 方法的处理逻辑。绑定操作，服务端在启动时调用bind方法时触发（手动调用bind）。
  - void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress,SocketAddress localAddress, ChannelPromise promise)
    连接操作，客户端启动时调用connect方法时触发（手动调用connect）。
  - void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) 
    断开连接操作（手动调用disconnect）
  - void close(ChannelHandlerContext ctx, ChannelPromise promise)
    关闭通道，手动调用Channel#close方法时触发。(手动调用close)
  - void deregister(ChannelHandlerContext ctx, ChannelPromise promise) 
    调用Channel#deregister时触发。（手动调用deregister)。
  - void read(ChannelHandlerContext ctx) throws Exception
    注册读事件，并不是触发网络读写事件。
  - void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception
    调用调用 Channel 的 write(底层 SocketChannel 的 write)时触发。
  - void flush(ChannelHandlerContext ctx) 
    调用调用Channel#flush(SocketChannel#flush)时触发。

- ChannelDuplexHandler
  双向 Handler，包含 Inbound 和 outbound 事件。

- ByteToMessageDecoder
  解码器：字节流解码成一条一条的消息(Message、协议对象)。

- MessageToByteEncoder
  编码器：消息（协议对象）编码成二进制字节流。

- AbstractTrafficShapingHandler
  流量整形，将在后续章节中详细介绍。

上述详细的介绍了NettyChannel的类继承体系，并重点介绍了ChannelInboundHandler 与 ChannelOutboundHandler 每个方法的含义已经触发时机

**ChannelInboundHandler**：**入端操作**，可以看出基本上是都是由事件选择器(NIO Selector事件就绪选择)进行触发,事件名称以 channel 开头，例如channelRead。

**ChannelOutboundHanlder**：**出端操作**，其触发点除了 read 事件外都是通过调用api(例如bind、connect、close、write)。

最后以一个思考题结束本文的讲解：ChannelInboundHandler 的**channelRead 事件**与 ChannelOutboundHandler 的 **read 事件**有什么区别呢？