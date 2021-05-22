title: kafka-01-原理
date: 2021-02-13 17:05:46
tags:

# **Kafka** 概述

Apache Kafka 是一个快速、可扩展的、高吞吐的、可容错的分布式“发布-订阅”消息系统， 使用 Scala 与 Java 语言编写，能够将消息从一个端点传递到另一个端点，较之传统的消息中 间件(例如 ActiveMQ、RabbitMQ)，Kafka 具有高吞吐量、内置分区、支持消息副本和高容 错的特性，非常适合大规模消息处理应用程序。

<img src="https://img-blog.csdnimg.cn/20201025100102289.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1cGVyaW9ycGVuZ0ZpZ2h0,size_16,color_FFFFFF,t_70" alt="img" style="zoom: 67%;" />

## 应用场景

- 用户的活动追踪

用户在网站的不同活动消息发布到不同的主题中心，然后可以对这些消息进行实时监测实时处理。当然，也可加载到 Hadoop 或离线处理数据仓库，对用户进行画像。像淘宝、京东这些大型的电商平台，用户的所有活动都是要进行追踪的。

- 日志聚合
  ![img](https://img-blog.csdnimg.cn/20201025100208271.png)

- 限流削峰
  ![img](https://img-blog.csdnimg.cn/20201025100319668.png)

## **kafka** 高吞吐率实现

Kafka 与其它 MQ 相比，其最大的特点就是高吞吐率。为了增加存储能力，Kafka 将所有 的消息都写入到了低速大容的硬盘。按理说，这将导致性能损失，但实际上，kafka 仍可保 持超高的吞吐率，性能并未受到影响。其主要采用了如下的方式实现了高吞吐率。

- 顺序读写:Kafka将消息写入到了分区partition中，而分区中消息是顺序读写的。顺序

  读写要远快于随机读写。

- 零拷贝:生产者、消费者对于kafka中消息的操作是采用零拷贝实现的，即不经过操作系统的ALU，不经过用户空间，消息直接写入内核。<!--见Ngnix中对零拷贝的讲解-->

- 批量发送:Kafka允许使用批量消息发送模式。

- 消息压缩:Kafka支持对消息集合进行压缩。



# **Kafka** 工作原理与工作过程

> 思考：
>
> 1. 如果集群中topic只有一个partition，那么如果client端连接上其他server，可以删除这个topic吗？
>
>    - 肯定是可以的。因为其他server会路由到Controller sever, 然后Controller server知道这个topic下的partition位于哪台server中。
>
> 2. 对于集群，哪些是client，哪些是server?区别是什么？
>
>    - consumer和producer都是client, 他们都不需要启动，只需要运行脚本文件就可以了。kafka集群是server，必须启动运行。
>
> 3. Kafka集群和Redis集群有什么区别？
>
>    - kafka集群是连接到zk上的，topic信息都是zk存储；Redis集群是自己管理，每个节点都会存储其他节点的信息。
>
>      

[kafka本身即是server，又是client，体现如下]()

> - 创建topic的时候，链接zookeeper的时候，需要启动kafka，表明现在kafka就是一个server
> - 向topic发送消息的时候，可以不启动kafka，直接使用bin/kafka-console-producer.sh --topic quickstart-events命令，表明现在kafka就是一个client
> - 从topic接收消息的时候，同样也可以不用启动kafka



## **Kafka** 基本术语

### **Topic**

主题。在 Kafka 中，使用一个类别属性来划分消息的所属类，划分消息的这个类称为 topic。 topic 相当于消息的分类标签，是一个逻辑概念。<!--其他MQ也有-->

位置：[tmp/kafka-logs/[topic]-[partition]]()

### **Partition**

分区。topic 中的消息被分割为一个或多个 partition，其是一个物理概念，对应到系统上 就是一个或若干个目录。<!--其他MQ没有-->

位置：[tmp/kafka-logs/[topic]-[partition]]()<!--如果是集群，每个broker的partition不同，是递增关系-->

数量：一般为broker的整数倍，否则负载不均衡

> 逻辑结构							  实际目录
>
> Topic=test
>
> ​				partitions   
>
> ​				0							broker0：tmp/kafka-logs/test-0
>
> ​				1							broker1：tmp/kafka-logs/test-1
>
> ​				2							broker2：tmp/kafka-logs/test-2

### **segment**

段。将 partition 进一步细分为了若干的 segment，每个 segment 文件的最大大小相等。两部分组成，“. Index” file 和 “.log” file, 分别对应的是segment index file and data file.

- .log file从 000.0000.log开始，数据满了，增加另一个log file。

位置：[tmp/kafka-logs/[topic]-[partition]]()目录下的[xxx（20位bit）.log]()文件和[xxx（20位bit）.index

<!--kafka顺序写，指的是partition有序-->

[20bit]()的意思是表示该segment之前有多少条消息，比如第一个segment名称为0000...000.log (有2500条)，那么下一个segment名称为000.0002500.log(1000条)

<img src="https://miro.medium.com/max/1608/0*SIY8Kt3Y74tysckC.png" alt="Image for post" style="zoom: 50%;" />

> 比如，一个consumer要消费，手里有消息id（000..170414）,如何在broker中找到该消息吗？
>
> - broker中的partition有大量segment，根据二分法查找这些segment中名称最接近000..170414的，000..170410.log
>
> - 那么offset = 4, 再通过000..170410.index的偏移量4与映射关系找到000..170414这个消息在磁盘中的位置。完成

### **Broker**

Kafka 集群包含多个服务器，每个服务器节点称为一个 broker。 一个 topic 中设置 partition 的数量是 broker 数量的整数倍。

### **Producer**

生产者。即消息的发布者，其会将某 topic 的消息发布到相应的 partition 中。

### **Consumer Group**

消费者。可以从 broker 中读取消息，其是以 consumer group 的形式出现的。

- consumer group 是 kafka 提供的可扩展且具有容错性的消费者机制。组内可以有多个消 费者，它们共享一个公共的 ID，即 group ID。[组内的所有消费者会协调在一起平均消费订阅主题的所有分区]()。

  Kafka 可以保证在稳定状态下，[一个 partition 中的消息只能被同一个 consumer group 中 的一个 consumer 消费，不能被多个consumer消费]()，而[一个组内 consumer 只会消费某一个或几个特定的 partition]()。当然， 一个消息可以同时被多个 consumer group 消费。

- 组中 consumer 数量与 partition 数量的对应关系如下。

  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in01.png" alt="4 partition, 1 consumer" style="zoom: 25%;" />

  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in02.png" alt="ktdg 04in02" style="zoom: 25%;" />

  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in03.png" alt="ktdg 04in03" style="zoom: 25%;" />
  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in04.png" alt="ktdg 04in04" style="zoom:25%;" />
  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in05.png" alt="ktdg 04in05" style="zoom:25%;" />

   这样设计的优点:实现简单、控制简单。不足:consumer 对于消息的消费不均衡，注意，不是对 partition 分配的不均衡。

### **Replicas of partition** 

分区副本。副本是一个分区的备份，是为了防止消息丢失而创建的分区的备份。

只允许挂replicas-1台机器

### **Partition Leader** 与 Partition Follower

- 每个 partition 有多个副本replicas，<!--这块我经常混淆，以为topic-partition-0、topic-partition-1....互为副本 --> 其中有且仅有一个作为 Leader，Leader 是当前负责消息读写 的 partition。[即所有读写操作只能发生于 Leader 分区上。](http://dpurl.cn/HxIrgGcz)
- 所有 Follower 都需要从 Leader 同步消息，Follower 与 Leader 始终保持消息同步。 partition leader 与 partition follower 是主备关系，不是主从。
  - 主备：主干活，从不干活。除非主挂了
  - 主从：主干活，从业干活，但是一般读写分离

### **ISR**

<img src="https://miro.medium.com/max/2074/0*EKINPKA_r9-9ip5k.png" alt="Image for post" style="zoom:67%;" />

ISR，In-Sync Replicas，是指partition副本同步列表。 
AR，Assiged Replicas，是指所有partition副本。 
OSR，Outof-Sync Replicas，是同步超时的partition副本。leader和follower通信超时，那么follower进入OSR
AR = ISR + OSR

### **offset**

偏移量。每条消息都有一个当前 Partition 下唯一的 64 字节的 offset，它是相对于当前 分区第一条消息的偏移量。

### **offset commit**

当 consumer 从 partition 中消费了消息后，consumer 会将其消费的消息的 offset 提交给 broker，表示当前 partition 已经消费到了该 offset 所标识的消息。

Consumer 从 partition 中取出一批消息写入到 buffer 对其进行消费，在规定时间内消费 完消息后，会自动将其消费消息的 offset 提交给 broker，以让 broker 记录下哪些消息是消费 过的。当然，若在时限内没有消费完毕，其是不会提交 offset 的。

### **Rebalance**

当消费者组中消费者数量发生变化，或 Topic 中的 partition 数量发生了变化时，partition 的所有权会在消费者间转移，即 partition 会重新分配，这个过程称为再均衡 Rebalance。

再均衡能够给消费者组及 broker 集群带来高可用性和伸缩性，但在再均衡期间消费者 是无法读取消息的，即整个 broker 集群有一小段时间是不可用的。因此要避免不必要的再 均衡。

### **__consumer_offsets**

消费者提交的 offset 被封装为了一种特殊的消息被写入到了一个由系统创建的、名称为 __consumer_offset 的特殊topic的 partitions 中了。该 Topic 默认包含 50 个分区。[在partition中的offset默认有效期为一天]()<!--如果两次消费中间隔了一天，那么offset就失效了，悲剧了，哈哈！不过，这种场景不会发生在生产中-->

每个 Consumer Group 的消费的 offset 都会被记录在__consumer_offsets 主题的 partition 中。该主题的 partition 默认有 50 个，那么 Consumer Group 消费的 offset 存放在哪个分区呢? 其有计算公式:[Math.abs(groupID.hashCode()) % 50]()。<!--也就是同一个groupID,永远用的同一个partition-->

写入到__consumer_offsets 主题的 partition 中的 offset 消息格式为:
<font color="color">[Group, Topic, Partition]::[OffsetMetadata[Offset, Metadata], CommitTime, ExpirationTime]</font>

- **Broker Controller**

  Kafka 集群的多个 broker 中，有一个会被选举为 controller，负责管理整个集群中 partition 和副本 replicas 的状态。

- **Zookeeper**

  Zookeeper 负责维护和协调 broker，负责 Broker Controller 的选举。 

- **Group Coordinator**

  group Coordinator 是运行在每个 broker 上的进程，主要用于 Consumer Group 中的各个 成员的 offset 位移管理和 Rebalance。Group Coordinator 同时管理着当前 broker 的所有消费 者组。

### **Broker Controller**

Kafka 集群的多个 broker 中，有一个会被选举为 controller，负责管理整个集群中 partition 和副本 replicas 的状态。

### Zookeeper

Zookeeper 负责维护和协调 broker，负责 Broker Controller 的选举。 总结:

- partition leader是broker controller选举出来的
- broker controller是zk选举出来的

### Group Coordinator

group Coordinator 是运行在每个 broker 上的进程，主要用于 Consumer Group 中的各个 成员的 offset 位移管理和 Rebalance。Group Coordinator 同时管理着当前 broker 的所有消费 者组。

> 当 Consumer 发送请求要消费数据时，当前 broker并不是直接从__consumer_offset 的 partition 中获取的， 而是从当前 broker 的 Coordinator 的缓存中获取的。<!--因为partition是在磁盘中，读取效率低-->

> 缓存中的数据从哪里来呢?
>
> 当 Consumer 消费完毕提交 offset 时，会同时提交到当前 broker的coordinator 的缓存__consumer_offset 的 partition。

## Kafka工作原理与过程

### 生产者消息路由策略

在通过 API 方式发布消息时，生产者是以 Record 为消息进行发布的。Record 中包含 key 与 value，value 才是我们真正的消息本身，而 key 用于路由消息所要存放的 Partition。消息 要写入到哪个 Partition 并不是随机的，而是有路由策略的。

1. 若指定了 partition，则直接写入到指定的 partition;

2. 若未指定 partition 但指定了 key，则通过对 key 的 hash 值与 partition 数量取模，该取模结果就是要选出的 partition 索引;

3. 若 partition 和 key 都未指定，则使用轮询算法选出一个 partition。

### 生产者消息写入算法

生产者将消息发送给 broker，并形成最终的可供消费者消费的 log，是一个比较复 杂的过程。

1) producer 向 broker 集群提交连接请求，其所连接上的任意 broker 都会向其发送 broker controller 的通信 URL，即 broker controller 主机配置文件中的 listeners 地址  <!--任意 broker为什么知道broker controller 的通信 URL呢？因为broker都是zk的客户端-->

2)  当 producer 指定了要生产消息的 topic 后，其会向 broker controller 发送请求，请求当前 topic 中所有 partition 的 leader 列表地址

3)  broker controller 在接收到请求后，会从 zk 中查找到指定 topic 的所有 partition 的 leader， 并返回给 producer

4)  producer 在接收到 leader 列表地址后，根据消息路由策略找到当前要发送消息所要发送 的 partition leader，然后将消息发送给该 leader

5)  leader 将消息写入本地 xxxx.log file，并通知 ISR 中的 followers

6)  ISR 中的 followers 从 leader 中同步消息后向 leader 发送 ACK

7)  leader 收到所有 ISR 中的 followers 的 ACK 后，增加 HW，表示消费者已经可以消费到该位置了

- 当然，若 leader 在等待的 followers 的 ACK 超时了，发现还有 follower 没有发送 ACK，则会将这些没有发送 ACK 的 follower 从 ISR 中清除，然后再增加 HW

### **HW** 机制（面试重点）

HW，HighWatermark，高水位，表示 Consumer 可以消费到的最高 partition 偏移量。HW 保证了 Kafka 集群中消息的一致性。确切地说，是在 broker [集群正常运转]()的状态下，保证了 partition 的 Follower 与 Leader 间数据的一致性。

LEO，Log End Offset，日志最后消息的偏移量。消息是被写入到 Kafka 的日志文件中的， 这是当前最后一个写入的消息在 Partition 中的偏移量。

对于 leader 新写入的消息，consumer 是不能立刻消费的。leader 会等待该消息被所有 ISR 中的 partition follower 同步后才会更新 HW，此时消息才能被 consumer 消费。

<font color='red'>就是消费者可以消费的消息，由HW决定的</font>

![Image for post](https://miro.medium.com/max/2096/0*lLqhkK6YUVb0LBr_.png)

### **HW** 截断机制

如果 partition leader 接收到了新的消息， ISR 中其它 Follower 正在同步过程中，[还未同步完毕时 leader 挂了,]()此时就需要选举出新的 leader。若没有 HW 截断机制，将会导致 partition 中 leader 与 follower 数据的不一致。

[当原 leader 宕机后又恢复时，将其 LEO 回退到其宕机时的 HW，然后再与新的 leader 进行数据同步，这种机制称为 HW 截断机制。]()

HW 截断机制可能会引发消息的丢失。

> 但是如何保证消息不丢呢？请看下面

### 生产者消息发送的可靠性机制

生产者向 kafka 发送消息时，可以选择需要的可靠性级别。通过 acks 参数的值进行设置。

(**1**) **0** 值

异步发送。生产者向 kafka 发送消息而不需要 kafka 反馈成功 ack。该方式效率最高，但可靠性最低。其可能会存在消息丢失的情况。

(**2**) **1** 值

同步发送，默认值。生产者发送消息给 kafka，broker 的 [partition leader 在收到消息后]()马上发送成功 ack(follower 是否同步完成，不影响 ack 的发送)，生产者收到后才会再发送 消息。如果一直未收到 kafka 的 ack，则生产者会认为消息发送失败，会重发消息。

- 该策略能否使生产者确认其发送的消息成功?不能。
  生产者即使收到了 ack，也不一定 就被 kafka 成功接收了。<!--比如leader收到后正在对follwers同步当中，宕机，然后根据[HW截断机制]()，该消息丢了。-->
- 该策略能否使生产者确认其发送的消息失败?可以。
  只要生产者没有收到 ack，消息发 送一定失败。

(**3**) **-1** 值

同步发送。其值等同于 all。生产者发送消息给 kafka，kafka 收到消息后要等到 [ISR 列表中的所有副本]()都同步消息完成后，[才向生产者发送成功 ack]()。如果一直未收到 kafka 的 ack， 则认为消息发送失败，会自动重发消息。

该模型可靠性最高，很少出现消息丢失的情况。<!--比如producer批量发送的时候，缓存满了，数据无法进入缓存，导致发送出差-->

但可能会出现部分 follower [重复接收]()消息的情况(不是[重复消费]())。<!--比如leader同步过程中宕机，则生产者没有收到ack，那么生产者重复数据，导致follower又接收一样的数据-->

### 消费者消费过程和offset

生产者将消息发送到 topic 中，消费者即可对其进行消费，其消费过程如下:

#### **消息重定向**

1)  consumer 向 broker 集群提交连接请求，其所连接上的任意 broker 都会向其发送 broker controller 的通信 URL，即 broker controller 主机配置文件中的 listeners 地址

2)  当 consumer 指定了要消费的 topic 后，其会向 broker controller 发送 poll 请求

3)  broker controller 会为 consumer 分配一个或几个 partition leader <!--比如3个broker有3个partition，那么有3个partition Leader;若有2个consumer，1个consumer分配一个Leader，另1个consumer分配2个Leader-->，并将该 partitioin 的当前 offset 发送给 consumer

>  a record gets delivered to only one consumer in a consumer group.

4)  consumer 会按照 broker controller 分配的 partition 对其中的消息进行消费



<img src="https://www.sderosiaux.com/static/044d8e05febd37679ba9b3c1bd82c1f2/00d43/consuming.png" alt="hop" style="zoom:67%;" />

#### **消费者确认收到**

5)  当消费者消费完该条消息后，消费者会向 broker 发送一个该消息已被消费的反馈，即该消息的 offset

#### 集群更新该group的offset

6)  当 broker 接到消费者的 offset 后，会更新到相应的__consumer_offset 中

>  one of the brokers is designated as the group’s **coordinator** and is responsible for managing the members of the group as well as their partition assignments. <!--就是说group的offset交给某一个partition的broker来管理，并且_consumer_offsets_partition作为特殊的topic，也是有leader partition（位于不同broker）的-->
>
> The coordinator of each group is chosen from the leaders of the internal offsets topic `__consumer_offsets`, which is used to store committed offsets. Basically [the group’s ID is hashed to one of the partitions for this topic]() and [the leader of that partition is selected as the coordinator](). In this way, management of consumer groups is divided roughly equally across all the brokers in the cluster, which allows the number of groups to scale by increasing the number of brokers.
>
> When the consumer starts up, it finds the coordinator for its group and sends a request to join the group. The coordinator then begins a group rebalance so that the new member is assigned its fair share of the group’s partitions. Every rebalance results in a new **generation** of the group.

> consumers groups each have their own offset per partition. 所以说有多少个partition，这个group就有多少个对应的offset
>
> the consumer groups have their own offset for every partition in the topic which is unique to what other consumer groups have.

#### 重复过程，并压缩日志

7)  以上过程一直重复，直到消费者停止请求消息

> kafka stores offset data in a topic called `"__consumer_offset" `. these topics use log compaction, which means they only save the most recent value per key. 就是类似于Redis的AOF压缩技术，只保存最新的value

8)  消费者可以重置 offset，从而可以灵活消费存储在 broker 上的消息



### 消费者重复消费问题及解决方案 

最常见的重复消费有两种:

(**1**) 同一个 **consumer** 重复消费

当 Consumer 由于消费能力较低而引发了消费超时时，则可能会形成重复消费。<!--比如consumer在1s内没有消费完指定的500条消息，consumer不会向broker提交offset,而是会发送error，那么broker会从原来的offset再发送原来的500条，导致consumer重复发送-->

解决方案:可以减少读取的消息个数，也可以延长自动提交的时间，还可以将自动提交转变为手动提交(consumer.commitSyn()方法)。

(**2**) 不同的 **consumer** 重复消费

当 Consumer 消费了消息但还未提交 offset 时宕机，则这些已被消费过的消息会被重复。<!--比如2个同组的consumer, 1st consumer挂了，未提交offset,那么rebalance后，2nd consumer会继续消费1st consumer原来对应partition中的数据-->

解决方案: 没办法绝对避免，可以通过对每个数据都要手动提交从而减少重复的数量

### **集群Partition Leader** 选举范围

当 leader 挂了后 broker controller 会从 ISR 中选一个 follower 成为新的 leader。但，若 ISR 中的所有副本都挂了怎么办?

可以通过 unclean.leader.election.enable 的取值来设置 Leader 选举的范围。

(**1**) **false**

必须等待 ISR 列表中有副本活过来才进行新的选举。该策略可靠性有保证，但可用性低。

(**2**) **true** 

在 ISR 中没有副本的情况下可以选择任何一个该 Topic 的 partition 作为新的 leader，该策略可用性高，但可靠性没有保证。可能会引发大量的消息丢失。<!--不推荐用,但是kafka用在日志系统比较多，所以丢了-->

## **kafka** 操作 

> [强调：kafka的操作，只要连到任意一个集群中的broker就行，最终都会路由的由controller，因为是zk管理的]()

### 启动/关闭kafka

```shell
bin/kafka-server-start.sh config/server.properties #启动kafka

bin/kafka-server-stop.sh  #关闭kafka
```



### 创建 **topic**

> partitions数量最好是与kafka server数量一致


```shell
$ bin/kafka-topics.sh --create --bootstrap-server 192.168.59.152:9092 --replication-factor 1 --partitions 1 --topic test_topic #分区数量是1，1个备份（replication）
```

```
ll tmp/logs  ##查看刚才创建的主题, 只有一个server有test_topic-0
```

partitions参数

- <font color="red">如果partitions < server，比如partitions=1, 那么只有一个server有test_topic，即test_topic-0</font>
- <font color="red">如果partitions = server，那么每个server都有test_topic，即test_topic-0, test_topic-1 ....</font>

replication参数

- <font color="red">如果replication=1，那么整个kafka集群，只有一个server含有test_topic的一个备份</font>

### 查看 **topic**

```shell
$ bin/kafka-topics.sh --list --bootstrap-server 192.168.59.153:9092 ##无所谓连接哪个bootstrap-server，因为是集群
```

### 发送消息 

该命令会创建一个生产者，然后由其生产消息,发送给topic。

> --bootstrap-server 随意一个kafka server，因为broker内部有controller，会将信息进行发送给topic

```shell
$ bin/kafka-console-producer.sh --topic topic_test --bootstrap-server 192.168.59.153:9092
```

### 消费消息

```bash
$ bin/kafka-console-consumer.sh --topic topic_test --from-beginning --bootstrap-server 192.168.59.153:9092
```

-  --from-beginning：表示从头开始消费，否则只能消费新的消息。

### 继续生产消费

### 删除 **topic**

> --bootstrap-server 随意一个kafka server，即便这个server没有存储该topic的partition，因为broker内部有controller，会将信息进行发送给controller，然后由controller定位到含有topic partition的server，进行删除

```shell
$ bin/kafka-topics.sh --delete --bootstrap-server 192.168.59.152:9092 --topic topic_test 
```

## zk操作

```shell
zkCli.sh #进入zk命令行
```

### 查看目录信息

```shell
ls /
[admin, brokers, cluster, config, consumers, controller, controller_epoch, feature, isr_change_notification, latest_producer_id_block, log_dir_event_notification, zookeeper]
```

### 查看kafka集群的broker

```shell
ls /
[admin, brokers, cluster, config, consumers, controller, controller_epoch, feature, isr_change_notification, latest_producer_id_block, log_dir_event_notification, zookeeper]

ls /brokers 
[ids, seqid, topics]

ls /brokers/ids #当时给broker配置的broker.id
[0, 1, 3]

get /brokers/ids/0 #查看broker.id=0的broker信息
{"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://192.168.199.134:9092"],"jmx_port":-1,"features":{},"host":"192.168.199.134","timestamp":"1615489355496","port":9092,"version":5}
```



### 查看kafka集群的topic

```shell
ls /brokers/topics

get /brokers/topics/cities/partitions/0 #可以看到isr、leader信息
```



### 删除topic

> rmr (deprecated) or deleteall

```shell
#找到要删除的topic，然后执行命令，此时topic被彻底删除
rmr /brokers/topics/【topic name】

#如果topic 是被标记为 marked for deletion，则通过命令 ls /admin/delete_topics，找到要删除的topic，然后执行命令：
rmr /admin/delete_topics/【topic name】
```



# 日志查看

我们这里说的日志不是 Kafka 的启动日志，启动日志在 Kafka 安装目录下的 logs/server.log 中。消息在磁盘上都是以日志的形式保存的。我们这里说的日志是存放在/tmp/kafka_logs 目录中的消息日志，即 partition 与 segment

## 查看topic的分区partition与备份Replicas

```
ls /tmp/kafka_logs
```

- 可以看到[topic-主机分区号]()，如果备份=主机数量，此主机还有[topic-其他主机分区号]()

### 1 **个分区** **1** **个备份**

我们前面创建的 test 主题是 1 个分区 1 个备份。[发现只有1个broker含有test-0]()

| broker  | topic-partition |
| ------- | --------------- |
| broker1 | test-0          |
| broker2 | null            |
| broker3 | null            |

### **3** **个分区** **1** 个备份

再次创建一个主题，命名为 one，创建三个分区，但仍为一个备份。 依次查看三台broker，可以看到[每台 broker 中都有一个 one 主题的分区，分别为one-0, one-1, one-2。]()

| broker  | topic-partition |
| ------- | --------------- |
| broker1 | one-0           |
| broker2 | one-1           |
| broker3 | one-2           |

### **3** **个分区** 2 个备份

再次创建一个主题，命名为 p3r2，创建三个分区，2个备份。依次查看三台 broker，可以看到每台 broker 中都有2份 two 主题的分区。

| broker  | topic-partition |
| ------- | --------------- |
| broker1 | p3r2-0, p3r2-1  |
| broker2 | p3r2-0, p3r2-2  |
| broker3 | p3r2-1, p3r2-2  |

### **3** **个分区** 3 个备份

再次创建一个主题，命名为 p3r3，创建三个分区，三个备份。依次查看三台 broker，可以看到每台 broker 中都有三份 two 主题的分区。

| broker  | topic-partition       |
| ------- | --------------------- |
| broker1 | p3r3-0, p3r3-1,p3r2-2 |
| broker2 | p3r3-0, p3r3-1,p3r2-2 |
| broker3 | p3r3-0, p3r3-1,p3r2-2 |

### 总结

- partition是尽可能的均匀分布在各个broker上，
- replica表示整个broker所有的partition都算上，对于每一个partition都有replica个备份

## 查看特殊topic的分区

```shell
ls /tmp/kafka_logs
```

- 可以看到[__consumer_offsets 0 到 consumer_offsets 50](),默认50个
- 一个用户的一个主题会被提交到一个[__consumer_offsets 分区]()中。使用主题字符串的 hash 值与 50 取模，结果即为分区索引。

## 查看段 **segment** 

进入任意一个/tmp/kafka_logs/[topic-分区号]()目录下

(**1**) **segment** 文件

segment 是一个逻辑概念，其由两类物理文件组成，分别为“.index”文件和“.log”文 件。“.log”文件中存放的是消息，而“.index”文件中存放的是“.log”文件中消息的索引。

(**2**) 查看 **segment
** 对于 segment 中的 log 文件，不能直接通过 cat 命令查找其内容，而是需要通过 kafka自带的一个工具查看。

```shell
bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files 
/tmp/kafka-logs/test-0/00000000000000000000.log --print-data-log #消息里面有个payload字段，就是具体的信息
```

