---
title: Netty应用-02-编写一个NIO服务端
date: 2021-05-12 00:09:05
tags:
---

## Netty进阶：手把手教你如何编写一个NIO服务端

建议带着如下问题开始本文的阅读：

- ServerBootstrap 的 option 与 childOption 分别有什么作用
- 服务端IO通道如何绑定事件链。
- ServerBootstrap 的 handler 方法与 childHandler 方法的区别又是什么？
- childHandler中的方法在服务端bind方法时会被调用吗？

## 1、Netty服务端启动示例

------

基于Netty的使用示例如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFt3s8sgSic9TWkpHZXA2X9h3yS0qLo9cuQasAL8WpFVc6siagDTVBZHX0woLle51SF3zIYnbge6MehQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


代码@1：创建主从多Reactor线程模型的Boss线程组，通常只需要设置一个线程，用于监听客户端的连接请求(OP_ACCEPT)。

代码@2：创建主从多Reactor线程模型的Work线程组，即IO线程组，默认为CPU核数的两倍。

代码@3：创建Netty服务端启动工具类ServerBootstrap。

代码@4：调用group方法设置主从线程组。

代码@5：设置通道的类型，服务端NIO通道类型 NioServerSocketChannel。

代码@6：通过option方法为通道服务端通道选项。

代码@7：通过chiildOption方法为IO通道设置选项。

代码@8：通过ChannelInitializer添加自定义的ChannelHandler，通常包括编码解码器、业务Handler。

代码@9：调用管道的addLast添加自定义编码解码器。

代码@10：调用bind方法绑定到服务端指定接口，绑定完成后则在指点端口上监听客户端的连接。

服务端的核心流程入口为bind方法，接下来我们将详细分析其实现原理，继续体会NIO编程技巧。

## 2、Netty服务端启动流程

------

通过跟踪其bind方法，最终将进入到AbstractBootstrap的doBind方法。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFt3s8sgSic9TWkpHZXA2X9h3N3QP5R1TafamibLBVRL77OVTdzO2KDn1Dclcq3KKhkGUENqEUBHHf9A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

其关键实现点：
代码@1：通过调用initAndRegister方法完成底层网络初始化与通道注册工作。

代码@2：如果初始化与注册工作已完成，则直接调用doBind0方法完成绑定操作。



代码@3：如果初始化与注册工作未完成，则通过regFuture（注册凭证）中添加监听器，等注册完成后再执行doBind0方法。

> 技巧提示：基于Future异步编程，在主线程中通过调用future.isDone方法判断异步方法是否已完成，如果未完成，通过在该凭证上添加监听器（事件回调），操作完成后执行回调逻辑。

从上面的方法来看服务端的绑定流程包**含初始化与绑定两个子流程**，接下来将分别深入探讨。

#### 2.1 通道初始化

基于NIO编程，需要先创建通道，然后将其注册到**事件选择器**，这个过程由 AbstractBootstrap 的 initAndRegister 方法实现。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFt3s8sgSic9TWkpHZXA2X9h3km0ib77Wdc4BXvvDSUniafj4olsic6DOS1g2r5Tktwiagg0EToOyOVSWyQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


实现的关键点如下：
代码@1：创建NIO服务端通道实现类NioServerSocketChannel的实例。

代码@2：调用 init 方法初始化通道。

代码@3：将通道注册到事件轮询器EventLoopGroup

代码@4：:如果通道已注册，但发生了错误则调用通道close方法回收相关资源，如果未注册成功，则强制清除通道占用的资源，特别是文件占用符。

关于通道的注册逻辑已经在[手把手教你如何编写一个NIO客户端](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247485763&idx=1&sn=8f37462e810b127a9382a450c88476a7&chksm=e8c3feb7dfb477a10a117331940c9baaa2842d7aa9d03cc0aeb3fbc82292b59c14a996c6934d&scene=21#wechat_redirect) 中已详细介绍，故接下来重点关注一下服务端通道的注册流程。

##### 2.1.1 服务端通道初始化流程

AbstractBootstrap 的 init 方法是一个抽象方法，具体有其子类实现：
服务端通道的初始化代码由ServerBootstrap的init方法。
Step1：首先将通过ServerBootstrap设置的选项与附加选项初始化到通道中。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFt3s8sgSic9TWkpHZXA2X9h3Szq6z8Fiax1vBTmvUSHllDYe9AbT5kSicpj5DuT2gW0jnTPwWARyFkUw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
Step2：**init 方法的关键点**：将 handler 方法设置的事件链，同时新增  **ServerBootstrapAcceptor** 事件处理方法加入到 NioServerSocketChannel 的事件链，但并没有把 childHandler 中添加的事件链添加到NioServerSocketChannel。

**读者朋友们，请停下来思考一下，为什么会这样？从现在可以肯定的是 handler 方法定义的事件处理方法将在与 NioServerSocketChannel 相关的事件发生时其作用。**

要解开这个谜题，我们有必要来看看 ServerBootstrapAcceptor 是如何工作的。

##### 2.1.2 ServerBootstrapAcceptor 详解

ServerBootstrapAcceptor 类图如下所示：

ServerBootstrapAcceptor方法只实现了inbound事件的channelRead事件。在详细探究它之前先看看属性：

- EventLoopGroup childGroup
  事件执行器组，ServerBootstrap设置的从Reactor线程组，即Work线程组。
- ChannelHandler childHandler
  ServerBootstrap#childHandler 设置的事件处理器，也就是用户定义的事件处理器。

接下来探究其 channelRead 方法的实现逻辑：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFt3s8sgSic9TWkpHZXA2X9h3rPSoqqosTfYJiam9mCO4oRagvIWvva7jpYJ0cgtA1gFXOU14iaDcmCMw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


Step1：channelRead竟然传入的是一个Channel，那这个Channel对象是NioSocketChannel吗？

是的，原来当 OP_ACCEPT 事件触发后，Server端会通过调用ServerSocketChannel 的 accept()方法，将返回一个 NioSocketChannel，读写操作的载体，在NIO中负责数据的读写。

Step2：将通过 childHandler 定义的事件处理器绑定到 NioSocketChannel。

最终完成 NioServerSocketChannel 与 NioSocketChannel 的初始化与事件绑定。

关于 NioSocketChannel 详细的初始化流程蕴含在 ChannelInitializer，其机制已经在 [手把手教你如何编写一个NIO客户端](http://mp.weixin.qq.com/s?__biz=MzIzNzgyMjYxOQ==&mid=2247485763&idx=1&sn=8f37462e810b127a9382a450c88476a7&chksm=e8c3feb7dfb477a10a117331940c9baaa2842d7aa9d03cc0aeb3fbc82292b59c14a996c6934d&scene=21#wechat_redirect) 中详细介绍。

#### 2.2 NIO绑定机制

在通道完成初始化与注册后，服务端需要进行端口绑定，由 AbstractBootstrap 的 doBind0 方法实现。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFt3s8sgSic9TWkpHZXA2X9h3duTyZpDPlBraEDUx6MGOh4LJ8zFLaLEt9Pdb3jLZEYWNjqxRFLIyBw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


bind 的核心实现最终是调用 Channel 的 bind 方法，最终由 AbstractChannel 类实现：


bind事件将传播，根据Netty事件传播机制，bind 属于 ChannelOutbound事件，最终将调用 HeadContext的bind方法，最终将调用Unsafe的bind方法，更加具体是调用 AbstractChannel的内部类AbstractUnsafe的bind方法，其代码如下所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFt3s8sgSic9TWkpHZXA2X9h3cticUBjyQMsIeVk6icEup6J5pNwxasWic28XjIGsicoFgicMTBx4Yzy8Q3Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


doBind 方法是一个抽象方法，NIO服务端的实现：NioServerSocketChannel。


即最终通过调用NIO底层NioServerSocketChannel 的 bind 方法完成服务端通道的绑定操作，即实现服务端在特定端口监听客户客户端的连接请求。

