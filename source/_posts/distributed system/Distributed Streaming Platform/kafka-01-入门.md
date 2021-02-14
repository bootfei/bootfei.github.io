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

- 零拷贝:生产者、消费者对于kafka中消息的操作是采用零拷贝实现的，即不经过操作系统的ALU，不经过用户空间，消息直接写入内核

- 批量发送:Kafka允许使用批量消息发送模式。

- 消息压缩:Kafka支持对消息集合进行压缩。



# **Kafka** 工作原理与工作过程

## Kafka基本原理

## Kafka工作原理与过程

[kafka本身即是server，又是client，体现如下]()

> - 创建topic的时候，链接zookeeper的时候，需要启动kafka，表明现在kafka就是一个server
> - 向topic发送消息的时候，可以不启动kafka，直接使用bin/kafka-console-producer.sh --topic quickstart-events命令，表明现在kafka就是一个client
> - 从topic接收消息的时候，同样也可以不用启动kafka

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

