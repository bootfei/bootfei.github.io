---
title: Netty应用-01-编写一个NIO客户端
date: 2021-05-12 00:08:57
tags:
---

## Netty进阶：手把手教你如何编写一个NIO客户端

Netty是一款非常优秀的网络编程框架，是对NIO的二次封装，本文将重点剖析Netty客户端的启动流程，深入底层了解如何使用NIO编程客户端。

请带着如下问题开始本文的阅读：

- Netty是如何将客户端的事件加入到事件链中？
- Netty客户端在启动时需要注册读事件？
- Netty客户端在启动时需要注册写事件？
- 如果让你基于NIO写一个客户端需要实现的关键点是什么呢？

**上面的问题也可以这样问：在使用NIO实现客户端时该注册哪些事件？**

本节将详细学习Netty4客户端的启动流程，我们从一个Netty客户端使用示例入手。

## 1、Netty 客户端示例

------

Netty客户端示例如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuunoUbLgwcBzibnehIqicJeuqIbtHXrEKuYme0iaP2wicdDdCj7gLUjDqxug/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


对上面的步骤一一说明如下：
代码@1：创建 NIO EventLoopGroup，NIO 事件轮询器。

代码@2：创建Bootstrap实例，Netty提供的客户端操作工具类。

代码@3：指定创建 Channel 类型，客户端是 NioSocketChannel。

代码@4：通过option方法设置网络相关属性，可多次调用。

代码@5：通过调用 handler 方法添加客户端的事件处理器链。具体使用匿名类 ChannelInitializer，事件处理器链中通常会包含**编码器**、**解码器**、**业务处理Handler**(业务处理的入口)。

代码@6：调用 connect 方法异步建立连接。通过调用sync()方法转同步调用，等连接成功建立后才返回。

代码@7：创建一个Close Future，并且同步等待关闭事件的到达。

代码@8：安全优雅的关闭事件线程组。在具体事件过程中并不会这么使用，而是会将连接进行缓存，例如使用一个Map按IP进行缓存，发往同一个IP的请求使用同一条连接，在需要断开连接时再调用方法。

> 温馨提示：上述只是Demo级的示例，后续会给出工业级的Netty使用实战。

## 2、Netty客户端启动流程

------

本文试图揭晓Netty客户端的启动流程，让大家从底层彻底掌握其实现细节，便于更好的应用Netty，特别是对应排障起到一个很好的技能储备。

Netty 客户端的启动流程入口：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuuGtD9yicjj9Apgo3JMzTu8OXZpJ3csn8Ia3y0FFPvjQ5Yg3ibsOntvp6Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


其最终调用其 doConnect 方法。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuukzHntFqlFEVOy1mUkSTIK7TcG44mt0UqLZFibbHqJdibQlH9QPOibqM4g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


代码@1：调用 initAndRegister 初始化和注册到事件选择器，可联想 NIO 中通道创建后需要注册读写事件，这里就是需要将通道注册到事件选择器，稍后会重点介绍。

代码@2：initAndRegister 采用了异步机制，更确切是采用了 Future模式。如果已经完成则直接调用 doConnect0 方法执行连接。

代码@3：否则在 Future 上注册事件监听器，等注册成功后再进行调用连接。

接下来会对上面两个重要步骤进行详细介绍，**这里给我们使用 Future 模式提供了很好的示例**。

#### 2.1 通道注册

客户端在连接之前需要先将通道注册到事件选择器(Selector)中，具体事项如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuu96sCXjjpByg3ZmRY81KghricpkjvgvgJJTZAnU2sFdTEpr6vxjyRxdw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

关键如下：
代码@1：调用 init 方法对通过进行初始化，稍后详细介绍。


代码@2：将通过注册到事件轮询器中。

##### 2.1.1 通道初始化

通道初始化其代码如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuukibf36QSMcReZ6CbX0aOtEgSX8FTYCO1nr0ugrExuAAEGkyn5vdoHGQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


代码@1：将用户定义的Handler加入到事件处理器链(ChannelPipeline)，这里非常关键：客户端在启动时我们是通过 Bootstrap 的 handler 方法设置事件链。
代码@2、3主要是设置网络选项、属性。

**思考题：Netty是如何将客户端定义的多个Handler加入到事件链中呢？**

这个问题其实的**本质**是用户自定义的 **ChannelInitializer 的 initChannel 方法**在什么时候调用。

要解开该问题，我们就需要来看一下 ChannelInitializer。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuutjmjZTAtq7XpicJ6ztAOoRlLoxGM5j9ctuxdfDpWVmmSHKUYjU7QNmA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


上面图的几个关键点：

- 将Handler 加入到 ChannelPipeline 时其 handlerAdded 方法会被调用，其逻辑是如果该通道已经被注册到事件处理器则调用 initChannel 方法。
- 通道被注册到事件选择器(Selector)时会调用。
- initChannel 方法支持幂等，保证一个通道的 initChannel 方法不会被多次调用。

##### 2.1.2 通道注册

通道注册的入口方法：SingleThreadEventLoop。

NIO 的最终入口为 AbstractChannel 的内部类 AbstractUnsafe中的register方法。

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)


**这里无不透漏着Netty一个非常重要的编程技巧：一个通道会持有一个事件选择器(EventLoop)，如果调用通道的方法的线程不是EventLoop，则会把任务提交到EventLoop中执行。**

接下来重点看一下register0 方法，非常关键的一个方法。

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)


代码@1：进行底层NIO层面的注册，最终调用 AbstractNio 的 doRegister。

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)


**注意：通道注册，其注册的OP为0，并没有关注任何的事件(读、写等)**，**那问题来了，读写等事件在什么时候注册呢？**

代码@2：触发 handlerAdd 事件，因为在介绍ChannelInitializer handlerAdd事件执行的条件是通道已注册，如果未注册会将任务先挂起，这里就是触发其执行。

代码@3：传播通道注册事件，即在调用底层 NIO 的 register 方法将通道注册到事件选择器(Selector)后传播注册事件。

代码@4：如果通道激活并且是首次注册，则传播通道**激活事件**。

代码@5：如果通道不是首次注册并且开启了**自动注册读事件**，则自动为通道注册读事件。beginRead 方法最终会调用 AbstractNioChannel 的如下方法，底层使用了NIO的 selectionKey.interestOps 方法。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuuhgn2sI9YdApR4o1WT2c0UyvzGBLv9LqJ54WyRJetMXeAeiaDAbrowcQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


通道的注册流程就介绍到这里了。

**在NIO中，如果不注册读事件，不会从底层网络中读取数据，则对应客户端来说就不会去处理响应包，相对服务端来说不会去解析请求包，也就是无法完成请求与响应，故在NIO编程中，通道必须注册读事件。**

注册的逻辑就基本完成了，那问题来了，虽然开启了自动注册读事件，但初次注册时只是传播了channelActive事件，那在何时会注册读事件呢？

原理在传播 channelActive 事件后，在开启了自动注册读事件后自动注册读事件，其代码在 DefaultChannelPipeline 中内部类 HeadContext 的 channelActive 方法，其代码截图如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuu8Lz4hVzQ676MkT1CZzib4LRqHdm3S8PDAep6Mfp6UoVyh7MREjqsDQw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

#### 2.2 连接服务端

在来看一下客户端的启动核心方法：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuukzHntFqlFEVOy1mUkSTIK7TcG44mt0UqLZFibbHqJdibQlH9QPOibqM4g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


在完成了通道的初始化并注册到事件选择器后，接下来就需要向服务端发起连接操作，最终调用 doConnect0 方法，其基本的调用链如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuuiaN4hxEP8B76JgFLic4OYNic7uf4hdqMFpRVmbsCE0icibXaZVG7VxuWnZw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


故接下来我们重点从 AbstractNioUnsafe 的 connect 方法进行深入追踪。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuuOM7YIBZ9fjMJovibS9IUF9Ve2ooOiaq81Xyto02iaO4UEJnPSLicH4gmZw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


上面的方法几个关键点如下：

- 获取通道是否激活。
- 调用内部的doConnect 完成连接，将调用底层NIO相关的方法，从该方法可以学习如何编写NIO的连接代码，稍后会详细介绍。
- 由于NIO都是非阻塞的，**doConnect** 方法返回后并不代表通道已成功连接到服务端，故需要开启一个定时任务进行跟踪(**连接超时时间**)。

接下来我们详细介绍一下如何使用NIO编写连接代码。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuuIA0CUoPH6tkicqoIqW3yWSRMLyN9EKQhqaPK11r1p6d6t9R0Wc2VKAQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


代码@1：客户端连接服务端，通常是采用随机端口，当如果需要绑定到指定端口，调用 SocketChannel 的 bind 方法即可。

代码@2：调用底层 NIO 的 connect 方法，注意该方法并不会阻塞，返回成功表示连接建立成功；返回 false 表示还在连接中。

代码@3：如果是连接中，则注册OP_CONNECT事件，等连接成功后会收到对应的事件。

注册 OP_CONNECT 事件后，在事件选择器 NioEventLoop 的事件处理流程中会专门处理 OP_CONNECT 事件：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuu1byJg9aRGsV8uuEq9p9u9sZegg8u8UwOibABvU0icnSibxibZ9TtxklIMQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

NIO模式代码最终调用 NioSocketChannel 的 doFinishConnect 方法：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuuAu3ae7kicTp28fOccmldTh3TLKexlvicEpODUFeW0iaXb2REjC0ZoVUNw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


该方法通道在非阻塞模式下，如果连接未成功建立会返回false，由于这里是基于事件选择的，是在连接成功创建后才会触发，故这里只是简单的进行校验连接是否成功建立。

最后通道成功建立，需要触发通道激活事件，见下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFsZ12ibulBFb18NWr5Nt8icuu0Z8HncfEkYG1ibNPllicrVLkYO20DqnOyNtjjGn54jSS4wV8YoV0Lbsg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)AbstractNioChannel#fulfillConnectPromise

