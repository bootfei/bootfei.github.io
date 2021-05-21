---
title: kafka-分析与调优
date: 2021-05-21 12:34:40
tags:
---

### Kafka报 IO Exception(many open files)

#### 问题分析

------

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVd71ajm8l4SQibTbQic73zsFR1OryAj6gpuVAfviaTAyQzKmMNK1y5et2w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


首先我们要能看懂Kafka-manager上的一些监控指标，topic列表中关于topic的信息项如下所示：

- Brokers Spread %
  该topic中队列在Broker中的使用率，例如集群中有5个broker，但topic只在４个broker上创建了队列，那使用率为80％。

- Brokers Skew %   
  topic的队列倾斜率。如果集群中存在5个broker节点，topic的总分区数量为4,副本因子为2，但这些队列只分布在其中的４台broker中。那topic的**broker使用率(Broker Spread)为80%**。
  众所周知，引入多节点的目的就是负载均衡，队列在broker中的分配自然是希望越均衡越好，期望每台broker上存储2个队列(副本因子为2，总共８个队列)，表示没有发生倾斜，如果一台broker中的存在３个队列，而另外一个broker上１个队列，那说明发生了倾斜，**计算公式为超过平均队列数的broker节点个数除以总所在Broker数量**，其Brokers Skew等于(1/3)=33%。  <!--队列就是：分区 + 副本-->

- Brokers Leader Skew %
  topic分区中Leader分区的倾斜率。在Kafka中，**只有分区的Leader节点具有读写权限**，真正影响读写性能的是Leader分区是否均衡，试想一下，如果一个topic有6个分区，但所有的Leader分区只分布在一两个Broker节点上，这个**topic的写入、读取性能将受到制约，这个值建议维持在0％**。

- Replicas
  副本数、副本因子，即一个分区数据存储的份数，该数值包含Leader分区。

- Under Replicated %
  没有跟上复制进度的副本比例，在Kafka的复制模型中，主分区负责读写，该复制组内的其他副本从主节点同步数据，如果跟不上主节点的复制进度，将被提出ISR，被剔除ISR的副本不具备选举Leader的资格，**这个数据如果长期或频繁高于0，说明集群一定出现了问题**。

Producer Message/Sec
消息发送实时TPS，通过JMX采集，需要在kafka-manager中开启如下参数：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVibB9l70MzWAEFyUMZIvia0bxQrJYlciaeCmPBFNwWjJI95TiaUV293sX5g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- Summed Recent Offsets
  该主题当前最大的消息偏移量。

经过对Topic列表观察，发现开发环境存在大量的topic都只有一个队列，并且都分布在第一节点上，其截图如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVRdFRpiaHaB1GsEt8uibID722MYlxAurdoJnTK6EcIKrtDzGZjkqL5vjA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)在这里插入图片描述


从界面上对应的指标：Brokers Spread即Broker的利用率只有３分之一，抽取几个数据量大的主题，判断其路由信息，得知都分布在第一个Broker节点上，这样就导致其中一个节点大量出现文章开头部分提到的错误：**Too many open files**。

#### 解决方案

------

##### 3.1 扩分区

问题定位出来了，由于Broker利用率不均匀，大量topic只创建了一个队列，并且还集中落到了第一个节点。

针对这种情况，首先想到的方案：扩分区。

##### 3.1.1 通过Kafka-manager

Step1：在Kafka-manager的topic列表，点击具体的topic，进入详情页面，点击[add Partitions]，如图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVPXLfzpo8iaZoYQarg7ws4SczYTMFxic6YCNvJoYFmLs4kLl6Qwad1Ekg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


Step2：点击增加分区，弹出如下框：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVfuicI2KEPBO13zgMaw4Qbrcoe8meRTz2vyCDkx4QeicY8k9bYHZxk6oQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


说明如下：

- Partitions
  扩容后的总分区个数，并不是本次增加的分区个数。
- Brokers
  分区需要分布的Broker，建议全选，充分利用整个集群的性能。

##### 3.1.2 运维命令

可以通过Kafka提供的kafka-topics命令，修改topic的分区，具体参考如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVzjZFWZ47UpkVb3iaNkWMHqenWIXsCBjuGW1T1e42cRiaWiby6sQf4lJzQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

> 温馨提示：对这些运维命令不熟悉没关系，基本都提供了--help

##### 3.2 分区移动

由于存在大量的只有一个分区的topic，并且这些topic都分布到了第一个节点，是不是可以将某些topic的分区移动到其他节点呢？

接下来介绍一下分区移动如何操作。

##### 3.2.1 kafka-manager

Step1：进入topic详情页面，点击[Generate Partition Assignments]，如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVm6ibJZh43iaVwh40jX6Fz3AsbnibricqkmBWbOHRVhdMH5UiaMRhicPQic3Hw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


Step2：进入页面后，选择需要迁移到的brroker，还可以改变topic的副本因子，最后点击[Generate Partition Assignments]，如下图所示：

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVa0Z8umQiaib5DRzibX2I0icvPyMdiaEm3Px6F8ERyGFy9cBziaQEwIN0QzGQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


Step3：点击完成后，此时只是生成了分区迁移计划，并没有真正的执行，需要点击[Reassign Parttions]按钮。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVAceM0RH9Ktq7URs2FpdA404icN4qExz8ib0u0kJiaFNCHHz9QcZqDMVHg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

##### 3.2.2 运维命令

Step1：首先我们需要准备需要执行迁移的topic信息，例如将如下信息保存在文件dw_test_kafka_040802-topics-to-move.json中。

```
{"topics":
    [
        {"topic":"dw_test_kafka_040802"}
    ],
    "version": 1
}
```

Step2：使用kafka提供的kafka-reassign-partitions.sh命令生成执行计划

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVAUy43eHQT4wodWDoyINRX7icKnHesVuh6korWVXqjf3VrtYBZGU9gTw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


上面的参数其实对照kafka-manager的图理解起来会更快，点出如下关键点：

- --broker-list
  分区需要分布的broker。如果多个，使用双引号，例如 "0,1,2"。
- --topics-to-move-json-file
  需要执行迁移的topic列表。
- --generate
  表示生成执行计划(并不真正执行)

执行成功后会输出当前的分区分布计划与新的执行计划，通常我们可以先将当前的执行计划存储到一个备份目录中，将新生成的计划存储到一个文件中。

Step3：使用kafka提供的kafka-reassign-partitions.sh命令执行分区迁移计划

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVianvMtHR60SJxxBeHga7AZ7IsibjuAtTQL02R7hmFRy8wWgVv49brWGA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)


其关键点如下：

- --reassignment-json-file
  指定上一步骤生成的执行计划。

执行成功过后输出Successfully，重分区是一个非常复杂的过程，命令执行完成后，并不会真正执行完成，可以通过查询主题的详细信息来判断是否真正迁移成功。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Wkp2azia4QFvox3eJ4x9luN4F8S7KTYgVDGxJwcvfIUpiaxrjI16DiaEL9FR3HzmvjzeoPAOGvLwZhRpDHqpjwpOA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

#### 进阶与架构思维

------

通过kafka-reassign-partitions.sh对分区进行迁移，会影响业务方的正常使用吗？即会影响消息的消费与发送吗？

我们需要对分区迁移的实现原理做进一步探究，本文暂不从源码角度详细剖析，只是举例阐述一下分区迁移的实现机制。

需求：一个TopicA的其中一个分区p0，分布在broker id为1,2,3上，目前要将其迁移到brokerId为4,5,6。

在介绍迁移过程之前，我们先定义三个变量：

- OAR
  迁移前分区的分布情况。
- RAR
  迁移后的分区分布情况
- AR
  当前运行过程中的分区分布情况

结合上述例子，其整个迁移步骤如下：

|      AR       |  Leader(ISR)   | 说明                                                         |
| :-----------: | :------------: | :----------------------------------------------------------- |
|    {1,2,3}    |    1{1,2,3}    |                                                              |
| {1,2,3,4,5,6} |    1{1,2,3}    | 首先基于RAR集合(迁移后的新broker)上创建对应的分区，并开始从Leader同步数据 |
| {1,2,3,4,5,6} | 1{1,2,3,4,5,6} | 新创建的副本追上主节点的进度，并进入ISR集合                  |
| {1,2,3,4,5,6} | 4{1,2,3,4,5,6} | 如果Leader不在RAR所在的集合中，则发起一次选举，将Leader变更为RAR中其中一台。 |
| {1,2,3,4,5,6} |    4{4,5,6}    | 将OAR中的副本状态设置为OfflineReplica(下线)，将其从ISR中剔除 |
|    {4,5,6}    |    4{4,5,6}    | 删除下线的副本，完成整个迁移操作                             |

从上面这个过程，只有在Leader选举期间会对消息发送、消息消费造成影响，但通过Zookeeper实现Leader选举可在秒级别响应，结合Kafka消息发送端的缓冲队列、重试机制，在理论上可以做到对业务无影响。