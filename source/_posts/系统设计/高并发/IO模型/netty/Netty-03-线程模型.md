---
title: Netty-03-线程模型
date: 2021-05-12 00:17:25
tags:
---

Netty的内核主要包括如下图三个部分：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsF1gf64cKdY4n9Ns5ZhFJNM463cnGRkfCAMxsTEVq83cKIjMBjmvH9eFIZ82xISsrbnYOpCqpaWg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


其各个核心模块主要的职责如下：

- 内存管理
  主要提高高效的内存管理，包含内存分配，内存回收。

- 网通通道
  复制网络通信，例如实现对NIO、OIO等底层JAVA API 的封装，简化网络编程模型。

- 线程模型

  提供高效的线程协作模型。

大家不妨回想一下在以往的面试的过程中，面试官通常会问：**Netty 的线程模型是什么？**

**主从多 Reactor 模型**，相信大家都能脱口而出，然后呢？就没有然后了？

线程模型在网络通信中主要解决什么样的问题？在 Netty 中又是如何解决的，Netty 的线程模型为什么如此高效？请容我慢慢道来。

> 温馨提示：为了保证文章观点的严谨性，将探究领域锁定在：Netty NIO 相关。

## 1、主从多 Reactor 模型

------

主从多 Reactor 模型是业界一种非常经典的线程编程模型，其原理图如下所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsF1gf64cKdY4n9Ns5ZhFJN7ia6NQNFDMWGYvyocsb2ibNdSzicYMV5aPeQice8I8d8HOXYq75rbMmsxQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


我们首先简单介绍一下上图中涉及的几个重要角色：

- Acceptor

  请求接收者，在实践时其职责类似服务器，并不真正负责连接请求的建立，而只将其请求委托 Main Reactor 线程池来实现，起到一个转发的作用。

- Main Reactor
  主 Reactor 线程组，主要**负责连接事件**，并将**IO读写请求转发到 SubReactor 线程池**。当然在一些需要对客户端进行权限控制等场景下，权限校验的职责可以放到 Main Reactor 线程池，即 **Main Reactor 也可以注册通道的读写事件**，读取客户端权限校验相关的数据包，执行权限验证，权限验证通过后再将2通道注册到IO线程。

- Sub Reactor 
  Main Reactor 通常监听客户端连接后会将通道的读写转发到 Sub Reactor 线程池中一个线程(负载均衡)，负责数据的读写。在 NIO 中 通常注册通道的读(OP_READ)、写事件(OP_WRITE)。

为了更加深刻的理解主从 Reactor 模型，我们来看一下网络通讯一般会包含哪些关键动作：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsF1gf64cKdY4n9Ns5ZhFJNQug9CLUnGKLnIJAfmsMicEA8MCdYzEVgt0LzegolZOpaQCAStpFno8A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


一个网络交互通常的几个步骤如下：

- 服务端启动，并在特定端口上监听，例如 web 应用的 80端口。
- 客户端发起TCP的三次握手，与服务端建立连接，这里以 NIO 为例，连接成功建立后会创建NioSocketChannel对象。
- 服务端通过 NioSocketChannel 从网卡中**读取数据**。
- 服务端根据**通信协议**从二进制流中**解码**出一个个请求。
- 根据请求，**执行对应的业务操作**，例如 Dubbo 服务端接受一个查询用户ID为1的用户信息。
- 将业务执行结果返回到客户端，通常涉及到**协议编码、压缩**等。

**线程模型需要解决的问题**：连接监听、网络读写、编码、解码、业务执行这些操作步骤如何运用**多线程编程**，提升性能。

主从多Reactor模型是如何解决上面的问题呢？

1. 连接建立（OP_ACCEPT）由 Main Reactor 线程池负责，创建NioSocketChannel后，将其转发给SubReactor。

2. SubReactor 线程池主要**负责网络的读写**（从网络中读字节流、将字节流发送到网络中），即注册OP_READ、OP_WRITE，并且**同一个通道会绑定一个SubReactor线程**。

3. 编码、解码、业务执行，则具体情况具体分析

   通常**编码、解码会放在IO线程中执行，而业务逻辑的执行通常会采用额外的线程池**，但不是绝对的，一个好的框架通常会使用参数来进行定制化选择，例如 ping、pong 这种心跳包，直接在 IO 线程中执行，无需再转发到业务线程池，避免线程切换开销。

> 温馨提示：在网络编程中，通常将用于网络读写的线程称为IO线程。

## 2、Netty 的线程模型

------

Netty的线程模型是基于主从多Reactor模型。

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)


Netty 中网络的连接事件(OP_ACCEPT)由Main Reactor 线程组实现，**即 Boss Group，通常只需设置一个线程**。

网络的**读写**操作由 Work Group ( Sub Reactor) 线程组来实现，线程的个数默认为 2 * CPU Core，**一个 Channel 绑定到其中一个 Work 线程，一个 Work 线程中可以绑定多个 Channel**。

在 Netty 中编码、解码等操作会被封装成一个一个事件处理器(ChannelHandler)，那这些 Handler 是在IO线程池中执行？

默认情况下ChannelHandler 是在 IO 线程中执行，那如何改变默认行为呢？其关键代码如下：

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)


**关键点**：在将事件处理器添加到事件链时可以指定在哪个线程池中执行，如果不指定则为**IO线程**中执行。

面试官：**通常业务操作会专门开辟一个线程池，那业务处理完成之后，如何将响应结果通过 IO 线程写入到网卡中呢？**

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsF1gf64cKdY4n9Ns5ZhFJNNJQKJgs5MUIP1iaV4w9e9ggLEy5NrNic2LF09iaDoNtWZibN21LAPhk8Og/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


业务线程调用 Channel 对象的 write 方法并不会立即写入网络，只是将数据放入一个待写入队列(缓存区)，然后IO线程每次执行事件选择后，会从待写入缓存区中获取写入任务，将数据真正写入到网络中，数据到达网卡之前会经过一系列的 Channel Handler(Netty事件传播机制)，最终写入网卡。

最后再来介绍一下 Netty 中 IO 线程的大体工作流程。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsF1gf64cKdY4n9Ns5ZhFJNBCyrTibyhk3ssC1ib2GUlqDJlJ9oYibgpY3r8Xv7olEM0UUR4Gb1NGmZw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


IO线程处理的关键点：

- 每一IO线程在执行上述操作时是串行执行的，即注册在一个 Selector(事件选择器)中的所有通道，**同一时间只有一个通道的事件被处理。**这也是为什么NIO应对大文件传输时不具备优势的根本原因。
- IO 线程在处理完所有就绪事件后，还会从任务队列(Task Queue)获取任务，例如上文中提到的业务线程在执行完业务后需要将返回结果写入网络，**Netty 中所有的网络读写操作只能在IO线程中真正获得运行**，故业务线程需要将带写入的响应结果封装成 Task，放入到 IO 线程任务队列中。
- 事件传播机制，可以参考笔者近期发布的文章：[Netty 事件传播机制详解](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247485491&idx=1&sn=3d7eb1d25fc178b5792fb0ca9bc2504f&chksm=e8c3ffc7dfb476d14d80820a505bcf0393bec0eac75a20b16fb330b7e86899d74e696b260ecf&scene=21#wechat_redirect)

## 3、总结

------

回到到主题，如果我们在面试过程中碰到面试官提问“Netty 的线程模型是什么？”时，我们应该可以从容应对了。

我觉得可以从如下几个方面进行展开。

1. Netty的线程模型基于主从多Reactor模型。通常由一个线程负责处理OP_ACCEPT事件，拥有 CPU 核数的两倍的IO线程处理读写事件。
2. 一个通道的IO操作会绑定在一个IO线程中，而一个IO线程可以注册多个通道。
3. 在一个网络通信中通常会包含网络数据读写，编码、解码、业务处理。默认情况下编码、解码等操作会在IO线程中运行，但也可以指定其他线程池。
4. 通常业务处理会单独开启业务线程池，但也可以进一步细化，例如心跳包可以直接在IO线程中处理，而需要再转发给业务线程池，避免线程切换。
5. 在一个IO线程中所有通道的事件是**串行处理**的。