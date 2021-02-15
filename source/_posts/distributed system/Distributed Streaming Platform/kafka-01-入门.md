---
title: kafka-01-入门
date: 2021-02-13 17:05:46
tags:
---

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



[kafka本身即是server，又是client，体现如下]()

> - 创建topic的时候，链接zookeeper的时候，需要启动kafka，表明现在kafka就是一个server
> - 向topic发送消息的时候，可以不启动kafka，直接使用bin/kafka-console-producer.sh --topic quickstart-events命令，表明现在kafka就是一个client
> - 从topic接收消息的时候，同样也可以不用启动kafka



## **Kafka** 基本术语

### **Topic**

主题。在 Kafka 中，使用一个类别属性来划分消息的所属类，划分消息的这个类称为 topic。 topic 相当于消息的分类标签，是一个逻辑概念。<!--其他MQ也有-->

位于tmp/logs，名称为[[topic]-[分区]]()

### **Partition**

分区。topic 中的消息被分割为一个或多个 partition，其是一个物理概念，对应到系统上 就是一个或若干个目录。<!--其他MQ没有-->

### **segment**

段。将 partition 进一步细分为了若干的 segment，每个 segment 文件的最大大小相等。

位于[tmp/logs/[topic]-[分区]]()目录下，

### **Broker**

Kafka 集群包含一个或多个服务器，每个服务器节点称为一个 broker。 一个 topic 中设置 partition 的数量是 broker 数量的整数倍。

### **Producer**

生产者。即消息的发布者，其会将某 topic 的消息发布到相应的 partition 中。

### **Consumer Group**

消费者。可以从 broker 中读取消息，其是以 consumer group 的形式出现的。

- consumer group 是 kafka 提供的可扩展且具有容错性的消费者机制。组内可以有多个消 费者，它们共享一个公共的 ID，即 group ID。[组内的所有消费者会协调在一起平均消费订阅主题的所有分区]()。

  Kafka 可以保证在稳定状态下，一个 partition 中的消息只能被同一个 consumer group 中 的一个 consumer 消费，而一个组内 consumer 只会消费某一个或几个特定的 partition。当然， 一个消息可以同时被多个 consumer group 消费。

- 组中 consumer 数量与 partition 数量的对应关系如下。

  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in01.png" alt="4 partition, 1 consumer" style="zoom: 25%;" />

  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in02.png" alt="ktdg 04in02" style="zoom: 25%;" />

  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in03.png" alt="ktdg 04in03" style="zoom: 25%;" />
  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in04.png" alt="ktdg 04in04" style="zoom:25%;" />
  - <img src="https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in05.png" alt="ktdg 04in05" style="zoom:25%;" />

  组内 consumer 与 partition 有数量关系是 1:n，partition 与组内 consumer 有关系是:1:1。 一旦 consumer 与 partition 的消费关系确立就不会发生变化，直到这个 consumer 挂了。 这样设计的优点:实现简单、控制简单。不足:consumer 对于消息的消费不均衡，注意，不是对 partition 分配的不均衡。

### **Replicas of partition** 

分区副本。副本是一个分区的备份，是为了防止消息丢失而创建的分区的备份。

### **Partition Leader**

每个 partition 有多个副本，其中有且仅有一个作为 Leader，Leader 是当前负责消息读写 的 partition。即所有读写操作只能发生于 Leader 分区上。

### **Partition Follower**

所有 Follower 都需要从 Leader 同步消息，Follower 与 Leader 始终保持消息同步。 partition leader 与 partition follower 是主备关系，不是主从。

### **ISR**

ISR，In-Sync Replicas，是指副本同步列表。 
AR，Assiged Replicas，是指所有副本。 
OSR，Outof-Sync Replicas
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

消费者提交的 offset 被封装为了一种特殊的消息被写入到了一个由系统创建的、名称为 __consumer_offset 的特殊主题的 partitions 中了。该 Topic 默认包含 50 个分区。

每个 Consumer Group 的消费的 offset 都会被记录在__consumer_offsets 主题的 partition 中。该主题的 partition 默认有 50 个，那么 Consumer Group 消费的 offset 存放在哪个分区呢? 其有计算公式:[Math.abs(groupID.hashCode()) % 50]()。

- **Broker Controller**

  Kafka 集群的多个 broker 中，有一个会被选举为 controller，负责管理整个集群中 partition 和副本 replicas 的状态。

- **Zookeeper**

  Zookeeper 负责维护和协调 broker，负责 Broker Controller 的选举。 

- **Group Coordinator**

  group Coordinator 是运行在每个 broker 上的进程，主要用于 Consumer Group 中的各个 成员的 offset 位移管理和 Rebalance。Group Coordinator 同时管理着当前 broker 的所有消费 者组。

## Kafka工作原理与过程

##  **Kafka** 集群搭建

 在生产环境中为了防止单点问题，Kafka 都是以集群方式出现的。下面要搭建一个 Kafka集群，包含三个 Kafka 主机，即三个 Broker。

### **Kafka** 的下载

### 安装并配置第一台主机

- 上传并解压。 将下载好的 Kafka 压缩包上传至 CentOS 虚拟机，并解压。

- 创建软链接。为了屏蔽版本信息和目录，使得操作简单

  ```shell
  ln -s kafka_2.11/ kafka
  ```

- 修改配置文件。 在 kafka 安装目录下有一个 config/server.properties 文件，修改该文件。

  ```properties
  broker.id=0  #集群中kafka的broker唯一标识，默认0
  listeners=PLAINTEXT://192.168.59.152:9092 #写当前主机IP和端口。broker之间通信用的。不配置也行，kafka会自动配置
  log.dirs=xxx
  num.partitions=1  #消费的主题是要存储在partition中的，如果不设立，那么默认存储在1 partition中
  zookeeper.connect=192.168.10.32:2181  #consumer和broker是zookeeper管理的
  ```

  > 以 kafkaOS1 为母机再克隆两台 Kafka 主机。在克隆完毕后，需要修改 server.properties中的 broker.id、listeners 与 advertised.listeners。id要改为不一样的（+1就行）,listeners同样也不一样（ip改为各自的主机ip）

### **kafka** 的启动与停止 

- 启动 **zookeeper**

  - ```shell
    zkServer.sh start
    ```

- 启动 kafka，

  - 在命令后添加-daemon 参数，可以使 kafka 以守护进程方式启动，即不占用窗口。

  - ```
    bin/kafka-server-start.sh -daemon config/server.properties
    ```

- 停止 **kafka**

  - 先停kafka，再停zk

  - ```
    bin/kafka-server-stop.sh
    ```



## **kafka** 操作 

(**1**) 创建 **topic**

> partitions数量最好是与kafka server数量一致


```shell
$ bin/kafka-topics.sh --create --bootstrap-server 192.168.59.152:9092 --replication-factor 1 partitions 1 --topic test_topic #分区数量是1，1个备份（replication）
```

```
ll tmp/logs  ##查看刚才创建的主题, 只有一个server有test_topic-0
```

但是如果partitions数量与server一致，那么每个server都有test_topic-x (x=0~server数量-1)

如果replication数量=x>1，那么整个kafka集群总共有test_topic为x个

(**2**) 查看 **topic**

```shell
$ bin/kafka-topics.sh --list --bootstrap-server 192.168.59.153:9092 ##无所谓连接哪个bootstrap-server，因为是集群
```

(**3**) 发送消息 该命令会创建一个生产者，然后由其生产消息,发送给topic。

> --bootstrap-server 随意一个kafka server，因为broker内部有controller，会将信息进行发送给topic

```shell
$ bin/kafka-console-producer.sh --topic topic_test --bootstrap-server 192.168.59.153:9092
```

(**4**) 消费消息

```bash
$ bin/kafka-console-consumer.sh --topic topic_test --from-beginning --bootstrap-server 192.168.59.153:9092
```

(5) 继续生产消费

(**6**) 删除 **topic**

```
$ bin/kafka-topics.shkafka-consumer-groups.sh --delete --bootstrap-server 192.168.59.152:9092
```



# 日志查看

我们这里说的日志不是 Kafka 的启动日志，启动日志在 Kafka 安装目录下的 logs/server.log 中。消息在磁盘上都是以日志的形式保存的。我们这里说的日志是存放在/tmp/kafka_logs 目录中的消息日志，即 partition 与 segment

## 查看topic与Replicas of partition备份

```
ls /tmp/kafka_logs
```

- 可以看到[topic-主机分区号]()，如果备份=主机数量，此主机还有[topic-其他主机分区号]()

## 查看partition分区

```shell
ls /tmp/kafka_logs
```

- 可以看到[__consumer_offsets 0->__consumer_offsets 50](),默认50个

## 查看段 **segment** 

进入任意一个/tmp/kafka_logs/[topic-分区号]()目录下

(**1**) **segment** 文件

segment 是一个逻辑概念，其由两类物理文件组成，分别为“.index”文件和“.log”文 件。“.log”文件中存放的是消息，而“.index”文件中存放的是“.log”文件中消息的索引。

(**2**) 查看 **segment
** 对于 segment 中的 log 文件，不能直接通过 cat 命令查找其内容，而是需要通过 kafka自带的一个工具查看。

```
bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files 
/tmp/kafka-logs/test-0/00000000000000000000.log --print-data-log
```

一个用户的一个主题会被提交到一个[__consumer_offsets 分区]()中。使用主题字符串的 hash 值与 50 取模，结果即为分区索引。