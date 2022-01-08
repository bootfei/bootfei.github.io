---
title: kafka-03-保障消息有序性
date: 2021-05-27 19:20:31
tags:
---

## 保证有序性

### 为什么要保证消息的顺序

订单有很多状态，比如：下单、支付、完成、撤销等，不可能`下单`的消息都没读取到，就先读取`支付`或`撤销`的消息吧，如果真的这样，数据产生错乱

### 如何保证消息顺序

#### 方案1：producer端只写固定的partition（有缺陷）

`kafka`的`topic`是无序的，但是一个`topic`包含多个`partition`，每个`partition`内部是有序的。

![Image](https://mmbiz.qpic.cn/mmbiz_png/uL371281oDHq22JkuhhbPicwggABFQWlqOzug1A48UnG15tonQ3wBkbB5yicVnHpydn5Hq2PwiawKTVj8Vk6DB7sQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

只要保证生产者写消息时，按照一定的规则写到同一个`partition`，不同的消费者读不同的`partition`的消息，就能保证生产和消费者消息的顺序。

同一个`商户编号`的消息写到同一个`partition`，`topic`中创建了`4`个`partition`，然后部署了`4`个消费者节点，构成`消费者组`，一个`partition`对应一个消费者节点。从理论上说，这套方案是能够保证消息顺序的。![Image](https://mmbiz.qpic.cn/mmbiz_png/uL371281oDHq22JkuhhbPicwggABFQWlqrza43TUBKePfwRPaSrtSJiclN6ibdof2qKwyhIfGoicR4Z6C2icQP5vWuA/640)

**优点**

- 保障了producer发送信息的时候，是有序的

**缺陷**

- 不能保障Topic的partition接收信息，是有序的。比如，公司在那段时间网络经常不稳定，业务接口时不时报超时，业务请求时不时会连不上数据库，某条数据丢失了；还有，因为网络原因，最早发送的信息，最晚被topic接收 <!--不过，可以通过topic发送ack解决-->

#### 方案2：producer端异步失败重试，consumer端做判断

`同步重试机制`在出现异常的情况，会严重影响消息消费者的消费速度，降低它的吞吐量。

如果用`异步重试机制`，处理失败的消息就得保存到`重试表`下来。

但有个新问题立马出现：**只存一条消息如何保证顺序？**

存一条消息的确无法保证顺序，假如：”下单“消息失败了，还没来得及异步重试。此时，”支付“消息被消费了，它肯定是不能被正常消费的。

这时有种更简单的方案浮出水面：消费者在处理消息时，先判断该`订单号`在`重试表`有没有数据，如果有则直接把当前消息保存到`重试表`。如果没有，则进行业务处理，如果出现异常，把该消息保存到`重试表`。

后来我们用`elastic-job`建立了`失败重试机制`，如果重试了`7`次后还是失败，则将该消息的状态标记为`失败`，发邮件通知开发人员。





## 消息挤压

### 挤压原因1：消息体过大

一次简单的消息从生产到消费过程，需要经过`2次网络IO`和`2次磁盘IO`。如果消息体过大，势必会增加IO的耗时，进而影响kafka生产和消费的速度。消费者速度太慢的结果，就会出现消息积压情况。

可以这样设计了：

1. 订单系统发送的消息体只用包含：id和状态等关键信息。
2. 后厨显示系统消费消息后，通过id调用订单系统的订单详情查询接口获取数据。
3. 后厨显示系统判断数据库中是否有该订单的数据，如果没有则入库，有则更新。

![图片](https://mmbiz.qpic.cn/mmbiz_png/uL371281oDHq22JkuhhbPicwggABFQWlqib2hB0qSIXBPjxAH1vZQbn97tcMKQBmWBDL1Rc1ytjLXoXcCPic301pQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



### 挤压原因2：路由规则不合理

不是所有`partition`上的消息都有积压，而是只有一个。

![图片](https://mmbiz.qpic.cn/mmbiz_png/uL371281oDHq22JkuhhbPicwggABFQWlqdaENvsBRclJSZ2zvNoXxfpDS9IgJvpM6icibHB8Y32Jt4khMicw7wictmw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

刚开始，我以为是消费那个`partition`消息的节点出了什么问题导致的。但是经过排查，没有发现任何异常。

发现，有几个商户的订单量特别大，刚好这几个商户被分到同一个`partition`，使得该`partition`的消息量比其他`partition`要多很多。

这时我们才意识到，发消息时按`商户编号`路由`partition`的规则不合理，可能会导致有些`partition`消息太多，消费者处理不过来，而有些`partition`却因为消息太少，消费者出现空闲的情况。

为了避免出现这种分配不均匀的情况，我们需要对发消息的路由规则做一下调整。

用订单号做路由相对更均匀，不会出现单个订单发消息次数特别多的情况。除非是遇到某个人一直加菜的情况，但是加菜是需要花钱的，所以其实同一个订单的消息数量并不多。调整后按`订单号`路由到不同的`partition`，同一个订单号的消息，每次到发到同一个`partition`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/uL371281oDHq22JkuhhbPicwggABFQWlqoJO69ia8Fv9p1uc0HEHaJcYg85VaBlsHm25ubexHCFmmicbVWsN6IACA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



### 如何处理消息挤压？

每个`partition`都积压了`十几万`的消息没有消费，比以往加压的消息数量增加了`几百倍`。

原来是有其他业务在JOB中批量发消息导致的问题导致。

**积压的这`十几万`的消息该如何处理呢？**

- 直接调大`partition`数量是不行的，历史消息已经存储到4个固定的`partition`，只有新增的消息才会到新的`partition`。我们重点需要处理的是已有的partition。
- 直接加`consumer`服务节点也不行，因为`kafka`允许同topic的多个`partition`被一个`consumer`消费，但不允许一个`partition`被同组的多个`consumer`消费，可能会造成资源浪费。

- 用多线程处理，改成了用`线程池`处理消息，核心线程和最大线程数都配置成了`50`。线程数是可以通过`zookeeper`动态调整的

<img src="https://mmbiz.qpic.cn/mmbiz_png/uL371281oDHq22JkuhhbPicwggABFQWlqt9gnrBjrNqmrxqjJlghuZszz8ibdGic6KbthGAdYX8yqkSibxFlH1ibFeA/640" alt="图片" style="zoom:50%;" />

顺便说一下，[对于要求严格保证消息顺序的场景，可以将线程池改成多个队列，每个队列用单线程处理]()。





## 重复消费



`kafka`消费消息时支持三种模式：

- at most once模式 最多一次。保证每一条消息commit成功之后，再进行消费处理。消息可能会丢失，但不会重复。
- at least once模式 至少一次。保证每一条消息处理成功之后，再进行commit。消息不会丢失，但可能会重复。
- exactly once模式 精确传递一次。将offset作为唯一id与消息同时处理，并且保证处理的原子性。消息只会处理一次，不丢失也不会重复。但这种方式很难做到。

`kafka`默认的模式是`at least once`，但这种模式可能会产生[重复消费]()的问题，所以我们的业务逻辑必须做[幂等设计]()。

而我们的业务场景保存数据时使用了`INSERT INTO ...ON DUPLICATE KEY UPDATE`语法，不存在时插入，存在时更新，是天然支持幂等性的。

## 多环境消费问题

我们当时线上环境分为：`pre`(预发布环境) 和 `prod`(生产环境)，两个环境共用同一个数据库，并且共用同一个kafka集群。

需要注意的是，在配置`kafka`的`topic`的时候，要加前缀用于区分不同环境。pre环境的以pre_开头，比如：pre_order，生产环境以prod_开头，比如：prod_order，防止消息在不同环境中串了。

但有次运维在`pre`环境切换节点，配置`topic`的时候，配错了，配成了`prod`的`topic`。刚好那天，我们有新功能上`pre`环境。结果悲剧了，`prod`的有些消息被`pre`环境的`consumer`消费了，而由于消息体做了调整，导致`pre`环境的`consumer`处理消息一直失败。

其结果是生产环境丢了部分消息。不过还好，最后生产环境消费者通过重置`offset`，重新读取了那一部分消息解决了问题，没有造成太大损失。