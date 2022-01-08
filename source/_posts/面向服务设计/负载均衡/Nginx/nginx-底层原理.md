---
title: nginx-底层原理
date: 2021-05-24 09:13:55
tags:
---

它的高性能正是由于其优秀的架构设计，其架构主要包括这几点：模块化设计、事件驱动架构、请求的多阶段异步处理、管理进程与多工作进程设计、内存池的设计，以下内容依次进行说明。

# [模块化设计](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

高度模块化的设计是 Nginx 的架构基础。在 Nginx 中，除了少量的核心代码，其他一切皆为模块。

所有模块间是分层次、分类别的，Nginx 官方共有五大类型的模块：核心模块、配置模块、事件模块、HTTP 模块、mail 模块。它们之间的关系如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfeeAedAFMFzSSbskk3v94esp6TQRXIEz9hxQjQnejMf0CSrcGGdLxCfkAkgAczk2UAJhbvXa5RROQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

在这 5 种模块中，配置模块和核心模块是与 Nginx 框架密切相关的。而事件模块则是 HTTP 模块和 mail 模块的基础。HTTP 模块和 mail 模块的“地位”类似，它们都是更关注于应用层面。

# [事件驱动架构](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

事件驱动架构，简单的说就是由一些事件发生源来产生事件，由事件收集器来收集、分发事件，然后由事件处理器来处理这些事件（事件处理器需要先在事件收集器里注册自己想处理的事件）。

对于 Nginx 服务器而言，一般由网卡、磁盘产生事件，Nginx 中的事件模块将负责事件的收集、分发操作；而所有的模块都可能是事件消费者，它们首先需要向事件模块注册感兴趣的事件类型，这样，在有事件产生时，事件模块会把事件分发到相应的模块中进行处理。

对于传统 web 服务器（如 Apache）而言，采用的所谓事件驱动往往局限在 TCP 连接建立、关闭事件上，一个连接建立以后，在其关闭之前的所有操作都不再是事件驱动，这时会退化成按顺序执行每个操作的批处理模式，这样每个请求在连接建立后都将始终占用着系统资源 <!--系统资源具体指的是什么？线程、cpu还是内存、还是链接数？--> ，直到关闭才会释放资源。这种请求占用着服务器资源等待处理的模式会造成服务器资源极大的浪费。如下图所示，传统 web 服务器往往把一个进程或线程作为时间消费者，当一个请求产生的事件被该进程处理时，直到这个请求处理结束时，进程资源都将被这一请求所占用。比较典型的例子如 Apache 同步阻塞的多进程模式就是这样的。

传统 web 服务器处理事件的简单模型（矩形代表进程）:![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfeeAedAFMFzSSbskk3v94esLn1b6MngY8ibB8QKjyuWGcYCnwmvxovTLC6DONiaDzicTMlJl8CRrv6bg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Nginx 采用事件驱动架构处理业务的方式与传统的 web 服务器是不同的。它不使用进程或者线程来作为事件消费者，所谓的事件消费者只能是某个模块。只有事件收集、分发器才有资格占用进程资源，它们会在分发某个事件时调用事件消费模块使用当前占用的进程资源，如下图所示，该图中列出了 5 个不同的事件，在事件收集、分发者进程的一次处理过程中，这 5 个事件按照顺序被收集后，将开始使用当前进程分发事件，从而调用相应的事件消费者来处理事件。当然，这种分发、调用也是有序的。

Nginx 处理事件的简单模型：![Image](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfeeAedAFMFzSSbskk3v94esawqIqZZKUnFGzPmaialCSMjX01Y7w1nUQVkwlrjliaick6Mka1NVzVN7Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

由上图可以看出，处理请求事件时，Nginx 的事件消费者只是被事件分发者进程短期调用而已，这种设计使得网络性能、用户感知的请求时延都得到了提升，每个用户的请求所产生的事件会及时响应，整个服务器的网络吞吐量都会由于事件的及时响应而增大。当然，这也带来一定的要求，即每个事件消费者都不能有阻塞行为，否则将会由于长时间占用事件分发者进程而导致其他事件得不到及时响应，Nginx 的非阻塞特性就是由于它的模块都是满足这个要求的。

# [请求的多阶段异步处理](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

多阶段异步处理请求与事件驱动架构是密切相关的，也就是说，请求的多阶段异步处理只能基于事件驱动架构实现。多阶段异步处理就是把一个请求的处理过程按照事件的触发方式划分为多个阶段，每个阶段都可以由事件收集、分发器来触发。

处理获取静态文件的 HTTP 请求时切分的阶段及各阶段的触发事件如下所示：![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

这个例子中，该请求大致分为 7 个阶段，这些阶段是可以重复发生的，因此，一个下载静态资源请求可能会由于请求数据过大，网速不稳定等因素而被分解为成百上千个上图所列出的阶段。

异步处理和多阶段是相辅相成的，只有把请求分为多个阶段，才有所谓的异步处理。当一个时间被分发到事件消费者中进行处理时，事件消费者处理完这个事件只相当于处理完 1 个请求的阶段。什么时候可以处理下一个阶段呢？这只能等待内核的通知，即当下一次事件出现时，epoll 等事件分发器将会获取到通知，然后去调用事件消费者进行处理。

# [管理进程、多工作进程设计](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

Nginx 在启动后，会有一个 master 进程和多个 worker 进程。master 进程主要用来管理worker 进程，包括接收来自外界的信号，向各 worker 进程发送信号，监控 worker 进程的运行状态以及启动 worker 进程。worker 进程是用来处理来自客户端的请求事件。多个 worker 进程之间是对等的，它们同等竞争来自客户端的请求，各进程互相独立，一个请求只能在一个 worker 进程中处理。worker 进程的个数是可以设置的，一般会设置与机器 CPU 核数一致，这里面的原因与事件处理模型有关。Nginx 的进程模型，可由下图来表示：

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

在服务器上查看 Nginx 进程：

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

这种设计带来以下优点：

1） 利用多核系统的并发处理能力

现代操作系统已经支持多核 CPU 架构，这使得多个进程可以分别占用不同的 CPU 核心来工作。Nginx 中所有的 worker 工作进程都是完全平等的。这提高了网络性能、降低了请求的时延。

2） 负载均衡

多个 worker 工作进程通过进程间通信来实现负载均衡，即一个请求到来时更容易被分配到负载较轻的 worker 工作进程中处理。这也在一定程度上提高了网络性能、降低了请求的时延。

3） 管理进程会负责监控工作进程的状态，并负责管理其行为

管理进程不会占用多少系统资源，它只是用来启动、停止、监控或使用其他行为来控制工作进程。首先，这提高了系统的可靠性，当 worker 进程出现问题时，管理进程可以启动新的工作进程来避免系统性能的下降。其次，管理进程支持 Nginx 服务运行中的程序升级、配置项修改等操作，这种设计使得动态可扩展性、动态定制性较容易实现。

# [内存池的设计](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

为了避免出现内存碎片，减少向操作系统申请内存的次数、降低各个模块的开发复杂度，Nginx 设计了简单的内存池，它的作用主要是把多次向系统申请内存的操作整合成一次，这大大减少了 CPU 资源的消耗，同时减少了内存碎片。

因此，通常每一个请求都有一个简易的独立内存池（如每个 TCP 连接都分配了一个内存池），而在请求结束时则会销毁整个内存池，把曾经分配的内存一次性归还给操作系统。这种设计大大提高了模块开发的简单些，因为在模块申请内存后不用关心它的释放问题；而且因为分配内存次数的减少使得请求执行的时延得到了降低。同时，通过减少内存碎片，提高了内存的有效利用率和系统可处理的并发连接数，从而增强了网络性能。

