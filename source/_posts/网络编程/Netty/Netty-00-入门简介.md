---
title: Netty-00-入门简介
date: 2021-06-14 20:44:36
tags:
---

### BIO vs  NIO vs AIO

**BIO: 同步阻塞式IO**  打个比喻您打了专车，在您没有到之前司机就在出发地等您上车。您上车之后司机专门送您到目的地。在这个例子中您扮演着IO中的网络事件，司机扮演着处理网络事件的线程。整个过程中您如果没有任何事件发生司机一直都在等待这就是同步阻塞﻿。

**NIO:同步非阻塞IO**  银行柜员在等待人办理银行业务，人们去银行后首先要到取号机上取号然后等待对应的柜台叫号。在这个例子中银行柜员扮演着selector，办理银行业务的人扮演者网络事件，而取号机扮演者register的作用，银行柜台就是channel。整个过程中当没有人来办业务时，柜员是可以去做其他事情这就是非阻塞。

**AIO:异步非阻塞IO**  您点外卖后就去忙其他的事情了，等骑手把外卖送达后打电话告诉您外卖放外卖柜子里了；您闲下来的时候去取外卖。在这个例子里您扮演着处理网络事件线的程，外卖是网络事件；在这个过程中您可以在外卖还没有送来的时候做些其他的事情，等外卖送达后只是向您发送了一个外卖送达的事件。这就是异步和非阻塞。﻿

通过网络服务端代码的编写来让我们直观感受一下他们的区别：

BIOServer

```
public class Server {  //定义一个循环接收客户端的Socket连接请求。初始化一个线程池对象  private static ExecutorService poolHandler = new ThreadPoolExecutor(5, 5,                120, TimeUnit.SECONDS, new ArrayBlockingQueue<>(10));    public static void main(String[] args) {        try {            // 注册端口            ServerSocket ss = new ServerSocket(9999);            while (true) {                Socket socket = ss.accept();                // 把Socket封装成一个任务对象交给线程池处理                Runnable target = new ServerRunnable(socket);                poolHandler.execute(target);            }        } catch (IOException e) {            e.printStackTrace();        }    }
    public static class ServerRunnable implements Runnable {      private Socket socket;      public ServerRunnable(Socket socket) {          this.socket = socket;      }      @Override      public void run() {          // 处理接收的客户端Socket通信需求          try {              InputStream is = socket.getInputStream();              BufferedReader br = new BufferedReader(new InputStreamReader(is));              String msg;              while ((msg = br.readLine()) != null) {
                //处理数据的粘包拆包                //对完整的数据包进行解码操作                //得到客户端消息                //触发各种统计类事件如心跳检测 信息统计                 //处理客户端的消息                //得到响应消息                //对响应消息进行编码              }          } catch (IOException e) {              e.printStackTrace();            //处理网络断开事件            //处理其他异常事件          }      }  }}
```

NIOServer

```
public class Server {    public static void main(String[] args) throws IOException {        //获取ServerSocketChannel        ServerSocketChannel ssChannel = ServerSocketChannel.open();        //设置非阻塞模式        ssChannel.configureBlocking(false);        //绑定端口        ssChannel.bind(new InetSocketAddress(9999));        //获取选择器        Selector selector = Selector.open();        //将ServerSocketChannel注册到选择器上，并且监听建立连接事件        ssChannel.register(selector, SelectionKey.OP_ACCEPT);        // 使用Selector选择器轮询已经就绪好的事件        while (selector.select() > 0) {            // 获取选择器就绪事件            Iterator<SelectionKey> it = selector.selectedKeys().iterator();            //遍历事件            while (it.hasNext()) {                SelectionKey sk = it.next();                //判断事件类型                if (sk.isAcceptable()) {                    // 获取客户端channel                    SocketChannel channel = ssChannel.accept();                    //切换非阻塞模式                    channel.configureBlocking(false);                    //将该channel注册到选择器上                    channel.register(selector, SelectionKey.OP_READ);                } else if (sk.isReadable()) {                    //获取channel                    SocketChannel sChannel = (SocketChannel) sk.channel();                    //读取网络数据                    ByteBuffer buf = ByteBuffer.allocate(1024);                    int len = 0;                    while ((len = sChannel.read(buf)) > 0) {                        //处理数据的粘包拆包                        //对完整的数据包进行解码操作                        //得到客户端消息                        //触发各种统计类事件如心跳检测 信息统计                     }                }else if(sk.isWritable()){                        //得到响应消息                        //对响应消息进行编码                }                // 15.取消选择键SelectionKey                it.remove();            }        }    }}
```

用Netty进行网络编程时代码是这样的：

NettyServer

```
public class NettyServer {    public static void main(String[] args) throws Exception{        //设置接受网络连接线程池        NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);        //设置处理网络除连接外所有事件线程的线程池        NioEventLoopGroup workerGroup = new NioEventLoopGroup();        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(bossGroup,workerGroup)                .channel(NioServerSocketChannel.class)//设置Channel类型                .option(ChannelOption.SO_BACKLOG,1024)                .handler(new ChannelInitializer<ServerSocketChannel>() {                    @Override                    protected void initChannel(ServerSocketChannel ch) throws Exception {                      //设置处理网络连接的Handler                        ch.pipeline().addLast("serverBindHandler",                         new NettyBindHandler(NettyTcpServer.this,serverStreamLifecycleListeners));                    }                })                .childHandler(new ChannelInitializer<NioSocketChannel>() {                    @Override                    protected void initChannel(NioSocketChannel ch) throws Exception {                        ch.pipeline()                                .addLast("protocolHandler", new NettyProtocolHandler())//设置编解码器                                .addLast("serverIdleHandler",                                        new IdleStateHandler(0, 0, serverIdleTimeInSeconds))//设置心跳检测                                .addLast("serverHandler",new NettyServerStreamHandler(NettyTcpServer.this, false,                                        serverStreamLifecycleListeners,                                          serverStreamMessageListeners));//设置业务处理逻辑                    }                });
        ChannelFuture channelFuture = serverBootstrap.bind(9000).sync();        channelFuture.channel().closeFuture().sync();    }}
```

BIO和NIO编程在网络事件发生后都需要进行处理数据的粘包拆包、对完整的数据包进行解码、触发各种统计类事件、对响应消息进行编码、监听各种网络异常、需要对底层网络和通信协议有一定的了解。﻿

用Netty编程时只需要设置Handler就能快速的进行业务开发而不用关心数据的读取及网络事件的分发处理.让开发者从底层网络通信中解放出来。

从这些对比中我们可以看出Netty开发网络程序要求低，使用者无需太多关心和业务无关的信息。



**定制能力强**

**通过ChannelHandler灵活扩展**

多数情况下我们自定义的Handler会有多个，那么他们是怎么进行协调合作的呢？接下一起探索handler之间的协调合作。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dSPBDsriaSyCRnSzWNxvdXt7mnxoSZvspbP5mBLq87byzGa2F5vN9IZyQ/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)﻿﻿

通过上图我们可以知道我们定义的handler在Netty里是通过双向链表进行关联的。﻿

介绍一下读网络事件发生后Handler事件流：

- 假如每个handler都会把读事件向下一个InboundHandler类型的节点进行传递，此时的调用链路为head->A->B->C->tail；
- 假如B业务handler处理数据后不把读事件继续向下传递,此时B可以在自己内部选择不向下一个节点传递读事件.此时调用链路变为head->A->B；

当服务端有数据需要写入时又会发生什么呢？

- 假如每个handler都会把读事件向下一个OutboundHandler类型的节点进行传递,当C业务handler发送响应数据时此时调用链路为C->B->head；
- 假如业务B是参数校验的的headler,当校验失败就响应客户端.此时调用的链路为B->head﻿；

我们可以看出Netty通过控制InboundHandler节点的调用来决定读事件响应链路；通过控制OutboundHandler节点的调用来决定写事件调用链路。

**Handler强大**

Netty通过内置多种Handler让你在不了解底层网络、通信协议、编解码的背景下也可进行网络应用程序。它都内置有哪些Handler呢？

### 编解码

当你通过Netty发送或者接受一个消息的时候，就将会发生一次数据转换。入站消息会被解码，从字节转换为另一种格式（比如java对象）。如果是出站消息，它会被编码成字节。

#### **TCP粘包拆包**

大多数基于Netty通信底层都会使用TCP进行的通信。TCP在发送数据流的时一定会把整条数据流单独发送吗？答案是否定的。TCP在发送数据包的时会存在拆包和粘包问题。什么是TCP的粘包和拆包呢？

**TCP拆包:** 产生的原因就是消息体太大了,一个数据包里只能发送消息体的一部分；

**TCP粘包:** 产生的原因就是有多条消息体需要发送,一个数据包可以存放多个消息体；

![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dS4WVoHfAiaHuiaKicXmq9lhADmwAek4OXvYd4H5X5eqZvRXicCZgiadKUic9Q/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

由于篇幅有限关于TCP拆包和粘包只是大致说明了一些。

#### **编解码Handler**

Netty提供了三种解码器来解决TCP的拆包和粘包,他们分别是：

- LineBasedFrameDecoder(回车换行分包);
- DelimiterBasedFrameDecoder(特殊分隔符分包);
- FixedLengthFrameDecoder(固定长度报文来分包)

此外Netty还提供了N：

- 编解码字符串的StringEncoder和StringDecoder;
- 用于处理HTTP协议编解码HttpObjectDecoder和HttpObjectEncoder;
- 用于处理protobuf编解码ProtobufVarint32FrameDecoder和ProtobufDecoder；

除了以上介绍的编解码Netty内部还有很多内置编解码器,当您使用Netty开发时如果有用到编解码的时候可以首先查询一下Netty内部是否有实现。

### 支持多种主流协议

从Netty源码包上可以看出Netty基本上覆盖了主流协议的编解码实现，如HTTP、Protobuf、WebSocket、二进制等主流协议。

#### ﻿﻿![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dS0AOkj10sEic1TOgOU3Nz3OibVExRDnHl1Gc5PqH7Qic7s4qukyA1y1ibFg/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

### idle(心跳检测)

当需要监听网络连接是否长时间没有数据交换（如发送心跳包、关闭连接等）就可以使用内置的IdleStateHandler。

IdleStateHandler接收三个参数：

- readerIdleTimeSeconds: 读取空闲时间，即从上次读取数据到现在的秒数。
- writerIdleTimeSeconds: 写入空闲时间，即从上次写入数据到现在的秒数。
- allIdleTimeSeconds: 读写都空闲的时间，即从上次读写数据到现在的秒数。

当达到设定的空闲时间阈值时，IdleStateHandler会触发对应的IdleState事件，这些事件包括READ_IDLE、WRITE_IDLE和ALL_IDLE。你可以通过实现ChannelInboundHandler的channelIdle方法来监听这些事件，并在事件发生时执行相应的操作。





**高性能**



### 一次网络通信都发生了什么

1.客户端确定需要发送的数据；

2.数据从程序到系统然后通过网卡发送；

3.服务端收到读事件后把数据从系统读取到应用中；

4.应用处理客户端信息；

### Netty怎么提高通信性能

- 数据在系统中的存储Netty使用**ByteBuf**内存池来减少申请内存耗时；
- 数据在用户态到内核态的传输Netty采用了**内存零拷贝**（减少用户态到内核态的两次拷贝）来**减少传输耗时；**

**![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dSQbtEAPTNMt5GICMyCvgGicuicHXZB4Htm75JHFaKs5xNgWE6JolgIGrQ/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)**

- ﻿﻿数据在网络中的传输  数据传输时间取决于数据传输速度、数据包数量。对于传输速度无法优化，Netty内置了许多**编码器**，可以选择对数据压缩比较好的解码器来减少数据包的数量，以此来减少耗时；
- 服务端对网络事件的处理Netty采用主次**Reactor多线程模型来加快对网络事件的处理**，传统的BIO不能支持太多网络连接以及对系统资源使用率比较低。传统的NIO当网络连接数超多时网络事件得不到快速响应,造成大量客户端进行重试。

﻿﻿![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dSzWEibh5E65kPBoibdTKprwakA6RRX2DVyVZHqkMK1vFLC7pQHPeNzN1g/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**reactor单线程模型：** 有一个线程负责处理所有的网络事件。

﻿![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dS1cl44RlPrA4W4061kHgS4FYgfJ3dHnnIjJOt1lHthlnUic4Wjz8rlow/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

**Reactor多线程模型：**有一个线程单独处理建立网络事件，另外一个线程负责处理其他的网络事件。

﻿﻿![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dSZFCLWyVgA4OVRlBGy6BuyQM6ON49RQNWdH8bQ1FEr7icFnseLYOqiabg/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

主次**Reactor多线程模型：**有一个线程单独处理建立网络事件,把建立网络连接放到线程池中的某一个线程中，这个线程负责大量网络连接的其他请求.

从reactor线程模型上我们可以看出主次Reactor多线程模型可以快速对大量的网络事件进行响应,因此也会缩短网络事件处理时间.

- 客户端消息处理上，Netty采用了无锁串行化设计思想结合volatile的大量使用;通过读写锁提升并发性能来大大缩短了消息处理时间。

我们可以看到Netty针对网络传输的各个节点都做到了尽可能的缩短时间，这也是Netty高性能的原因所在。

Netty运行原理

通过以上对Netty是什么，优势有哪些已经有了初步了解。那么它内部是怎么工作的呢？接下来让我们一起看一下它内部的原理。





**整体结构**

![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dS2zydhafxsdJySlGIC2LncGBcxCz9ibliaso4lIrr8KRmwzZJBjAuiaic4w/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

上面这张图就是在官网首页的架构图，我们从上到下分析一下。

- **Core**核心层：核心层里Netty最精华的部分，它提供了底层网络通信的通用抽象和实现，包括事件模型、通用API、支持零拷贝的ByteBuf 等；
- **Protocol Support 协议支持层**：协议支持层基本上覆盖了主流协议的编解码实现，Netty 丰富的协议支持降低了用户的开发成本；
- **Transport Service 传输服务层：**传输服务层提供了网络传输能力的定义和实现方法。它支持 Socket、HTTP 隧道等传输方式。Netty 对 TCP、UDP 等数据传输做了抽象和封装，让开发者可以更聚焦在业务逻辑实现上，而不必关系底层数据传输的细节；

以上可看出Netty的功能、协议、传输方式都比较全，比较强大。





**逻辑架构**



![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dS72UajWyyFsVEePGL9BWFY9otrysdlXUNbhaRUJFF3mb3CCVfVgHgWA/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

从图中可以Netty通过网络通信层、事件调度层、服务编排层协调合作来运行程序

### 网络通信层

网络通信层的职责是执行网络 I/O 的操作。当网络数据读取到内核缓冲区后，会触发各种网络事件。这些网络事件会分发给事件调度层进行处理。接下来分别看一下Netty服务端和客户端在网络通信层是怎么运行的：

- Netty服务端：程序启动时会生成一个ServerBootstrap对象，该对象会生成一个NioServerSocketChannel来监听某一个端口的建立网络连接的事件，当网络连接建立后会监听网络连接的各种事件并通知到事件调度层。
- ﻿Netty客户端：程序启动的时会生成一个Bootstrap对象，该对象会生成一个NioSocketChannel来与服务端建立网络连接的事件，当网络连接建立后会监听网络连接的各种事件并通知到事件调度层。

### 事件调度层

事件调度层的职责是通过 Reactor 线程模型对各类事件进行聚合处理，通过 Selector 主循环线程集成多种事件(I/O 事件,信号事件,定时事件等)，实际的业务处理逻辑是交由服务编排层中相关的 Handler 完成。事件调度层主要由EventLoopGroup和EventLoop构成。﻿

- EventLoop 负责处理 I/O 事件和调度任务。每一个NioEventLoop内部都有唯一一个Selector，通过这个Selector可以对注册的channel进行网络事件的读取；NioEventLoop 还有一个内部的任务队列，可以用来提交 Runnable 任务。这些任务会在 NioEventLoop 的线程上下文中执行，确保了任务的顺序执行。
- EventLoopGroup  本质是一个线程池，负责管理EventLoop。其主要作用有从线程池挑选一个EventLoop进行channe的注册或者提交一个任务、关闭不再使用的 Channel、释放 Selector 和其他相关资源确保应用程序的干净退出。

### 服务编排层

服务编排层的职责是通过组装各类handler来实现网络数据流的处理。它是 Netty 的核心处理链，用以实现网络事件的动态编排和有序传播。

服务编排层的核心组件包括 **ChannelHandler、ChannelHandlerContext、ChannelPipeline**。他们之间的关系如下图所示：

﻿﻿![Image](https://mmbiz.qpic.cn/mmbiz_png/Z6bicxIx5naIrkrXjImG2icPTKeiangL9dS3EkyiaKtmgjichNyttyDSG1OtHFibic2DKmKI1ibfP1FgJtGiba9cXzxibdgg/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

- ﻿**ChannelHandler**主要分为两类，InboundHandler和OutboundHandler

- InboundHandler用于处理从网络流入(inbound)的数据和事件。帮助我们处理接收的数据、连接建立、断开等事件。核心方法有channelActive(网络连接建立成功时会调用)、channelInactive(网络连接断连时调用)、channelRead(接受到数据时调用)、exceptionCaught(如果在处理事件或数据时发生异常，该方法会被调用。可以在这里捕获并处理异常，防止应用程序崩溃。)

- OutboundHandler用于处理从应用程序流向网络的出站(outbound)操作,如写入数据,发起连接,关闭连接等核心方法有。

  write(写入数据到channel时调用),flush(将缓冲区所有未写入数据立即发送出去),connect(尝试建立网络连接时调用)。

- ﻿**ChannelHandlerContext**是handler与Netty内部机制交互的主要方式，它使得 handler 可以在不直接访问其他handler的情况下，协同处理I/O事件和数据。同时也是每个ChannelHandler在处理事件时的上下文环境，可以获取到Pipeline、Channel、Allocator等对象。
- ﻿**ChannelPipeline  Netty**中的关键组件，是一个处理网络I/O事件和数据的有序链表。ChannelPipeline负责将入站（inbound）和出站（outbound）事件分发给链中的各个ChannelHandler，实现了事件驱动的网络编程模型。每个ChannelHandler都有一个唯一的ChannelHandlerContext，用于与ChannelPipeline交互。





**运行流程**



我们已经从宏观上了解了Netty，接下来我们从服务端的视角简要的看一下Netty整个的运行流程。

1.服务端启动的时把ServerSocketChannel注册到boss EventLoopGroup中某一个EventLoop上，暂时把这个EventLoop叫做server EventLoop；

2.当 serverEventLoop中监听到有建立网络连接的事件后会把底层的SocketChannel和serverSocketChannel封装成为NioSocketChannel；

3.开始把自定义的ChannelHandler加载到NioSocketChannel 里的pipeline中，然后把该NioSocketChannel注册到worker EventLoopGroup中某一个EventLoop上，暂时把这个EventLoop叫做worker  EventLoop；

4.worker  EventLoop开始监听NioSocketChannel上所有网络事件；

5.当有读事件后就会调用pipeline中第一个InboundHandler的channelRead方法进行处理；

# 参考文献

1. Nettry入门：https://mp.weixin.qq.com/s/52iS3RxIO_to29IG_JoOpw