---
title: Kafka-06-高可用抛弃zk而用raft
date: 2021-09-07 18:39:48
tags:
---



## Zookeeper使用场景

使用Zookeeper提供的临时顺序节点与事件监听机制，能非常轻松的实现Leader选举。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFv5bvEeVf3VA9XJvdAvDTmibpuJPBnuUW0mcdicEkwfJPKr6o7hRicC86YvLFlBceVdZ6AULJS0iaEh4g/640)

正如上图所示，

- **客户端集群**的多个成员t1和t2，能提供相同的服务，但为了实现冷备效果(即同一时间只有一个成员对外提供服务，我们称之为Leader，当Leader宕机或停止服务后，该组织中的其他成名重新竞争Leader，然后继续对外提供服务)。

- **Zookeeper是注册中心，以集群部署的，能有效避免单点故障，并且集群内部提供了对数据的强一致性**。

[成为leader的步骤]()

```text
- 当客户端集群的成员需要竞争Leader时，借助Zookeeper的实现套路是向zookeeper中的一个数据节点(示例中为/app/order-service/leader)节点创建两个子节点，并且是**顺序的临时节点**。

- 判断客户端集群的成员判断创建的节点的序号是否为/app/order-service/leader中序号最小的节点，
  - 如果是则成为Leader，对外提供服务；
  - 如果序号不是最小的，则向自己前置的**注册节点**删除事件**，一旦Leader代表的进程宕机，它与Zookeeper的会话失效后，与之关联的临时节点会被删除，一旦Leader创建的节点被删除，其后继节点会得到通知，从而再次触发选主，选举出新的Leader，继续对外提供服务，保质服务的高可用性。
```

zk的选举机制特点

```text
  - 临时节点
    临时节点是与会话关联的，一点创建该临时节点的会话结束，与之会被自动删除，无需应用方人工删除。
  - 顺序节点
  - 事件机制
    借助与事件机制，Zookeeper能及时通知存活的其他应用节点，重新触发选举，使得实现自动主从切换变的非常简单。
```

  

## Kafka对Zookeeper的使用

Kafka中存在众多的Leader选举，一个topic可以拥有多个patition，每一个partition可以配置多个replica，**如何保证一个patition的数据在此patition的多个replica之间的一致性？**。

Kafka的实现就是一个partition有多个冗余(多个副本)，从中选举出一个Leader用来承担客户端的读写请求，从节点从主节点处拷贝内容，Leader节点根据数据在副本中成功写入情况，进行抉择来确定是否写入成功。<!--主备，不是主从-->

**故此处需要进行Leader选举**,而基于Zookeeper能轻松实现。

[![img](https://colobu.com/2017/11/02/kafka-replication/kafka_replication_diagram.png)](https://colobu.com/2017/11/02/kafka-replication/kafka_replication_diagram.png)

kafka的复制是针对分区的。比如上图中有4个broker, 1个topic, 2个分区，复制因子是3。当producer发送一个消息的时候，它会选择一个分区，比如`topic1-part1`分区，将消息发送给这个分区的leader， broker2、broker3会拉取这个消息，一旦消息被拉取过来，slave会发送ack给master，这时候master才commit这个log。




## Zookeeper的致命弱点

Zookeeper是集群部署，只要集群中超过半数节点存活，即可提供服务

但Zookeeper的设计是CP模型。Zookeeper集群中也存在所谓的Leader节点和从节点，Leader节点负责写，Leader与从节点可用接受读请求，但在Zookeeper内部节点在选举时整个Zookeeper无法对外提供服务。当然正常情况下选举会非常快，但在异常情况下就不好说了，例如Zookeeper节点发生full Gc，此时造成的影响将是毁灭性的。

Zookeeper节点如果频繁发生Full Gc，此时与客户端的会话将超时，由于此时无法响应客户端的心跳请求(Stop World)，从而与会话相关联的临时节点将被删除，注意，此时是所有的临时节点会被删除，Zookeeper依赖的事件通知机制将失效，整个集群的选举服务将失效。

站在高可用性的角度，Kafka集群的可用性不仅取决于自身，还受到了外部组件的制约，从长久来看，**显然都不是一个优雅的方案**。

**随着分布式领域相关技术的不断完善，去中心化的思想逐步兴起，去Zookeeper的呼声也越来越高，在这个进程中涌现了一个非常优秀的算法：Raft协议。**

Raft协议的两个重要组成部分：Leader选举、日志复制，而日志复制为多个副本提供数据强一致性提供了强一致性，**并且一个显著的特点是Raft节点是去中心化的架构，不依赖外部的组件，而是作为一个协议簇嵌入到应用中的，即与应用本身是融合为一体的**。

再以Kafka Topic的分布图举例，引用Raft协议的示例图如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFv5bvEeVf3VA9XJvdAvDTmibaH526cQCXlKkCHf9zf6wVOBIjic6vFjYYVZ7lEpHVPxv8kic2tbbRicMw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

选主提供了另外一种可行方案，而且还无需依赖第三方组件，故最终Kafka在2.8版本中正式废弃了Zookeeper，拥抱Raft。