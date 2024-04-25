---
title: '第1章:MySQL架构与历史'
date: 2020-12-15 09:28:13
tags: [db,mysql]
---

https://cxis.me/2018/05/01/Kafka%E6%A6%82%E8%A6%81/



Kafka设计初衷是为了解决互联网公司超大量级数据的实时传输，需要考虑以下问题：

- 吞吐量、延时，每次写操作会把数据写到OS的页缓存中，OS负责刷盘，Kafka采用追加方式，避免磁盘随机写操作；消费的时候读取消息会尝试从OS的页缓存中读取，命中缓存后直接发送到Socket上，就是零拷贝技术。
- 消息持久化
- 负载均衡、故障转移
- 伸缩性

# 基本概念

## 框架：

- Zookeeper集群，kafka用Zookeeper来负责集群元数据管理、控制器的选举等操作
- Producer，生产者，负责创建消息，投递到Kafka中
- Consumer，消费者，消费消息，连接到Kafka接收消息
- Broker，服务代理节点，负责存储消息

## 消息：

- Topic，消息以主题为单位归类，是一个逻辑概念，代表一类消息，通常可使用topic来区分实际业务。 topic通常会被多个消费者订阅。

## patition保证无状态的可拓展性：

- Partition，分区，主题是逻辑概念，可以细分为多个分区，一个分区只属于单个主题。同一主题下的不同分区包含消息是不同的，partition是不可修改的有序消息序列。用户对partition唯一能做的操作就是在消息序列尾部追加写入消息。partition上的每条消息都会被分配一个唯一的序列号，叫做位移offset
- Offset，topic的partition下每条消息都被分配一个唯一的offset，消费端也有offset概念，用来表示消费partition的消费进度。

## patition保证可用性

- Replica，Kafka的备份日志称为副本replica，防止数据丢失。副本分为两类：领导者副本leader replica和追随者副本follower replica。follower replica不负责响应客户端发来的消息写入和消费者请求，只是被动的向leader replica获取数据。leader replica所在broker宕机时，Kafka会从剩余的replica中选举新的leader继续提供服务

## patition保证一致性：

- Leader，leader对外提供服务
- Follower，follower被动追随leader状态，保持与leader同步，当做leader后备。leader挂掉后就会有一个follower备选举成新的leader。
- ISR，ISR是in-sync replica，与leader replica保持同步的replica集合。Kafka为partition动态维护一个replica，该集合中所有replica保存的消息日志都与leader replica保持同步，只有这个集合中的replica才能被选举为leader，只有该集合中所有replica都接到了同一条消息，Kafka才会认为消息是已提交状态，也就是消息发送成功。如果replica落后于leader replica的进度，当达到一定程度时，Kaffka会将这些replica踢出ISR，当这些replica追上了leader进度时，Kafka会将他们加入到ISR中。
- HW，Hight Watermark，高水位，标识了一个特定的消息偏移量，消费者只能拉取到这个offset之前的消息
- LEO，Log End Offset，标识当前日志文件中下一条待写入消息的offset

## 消息格式

消息由消息头部、key、value组成：

- CRC，4字节
- 版本号，1字节
- 属性，1字节
- 时间戳，8字节
- key长度，4字节
- key，k个字节，消息键，对消息做partition时使用，决定消息保存在某个topic下的哪个partition
- value长度，4字节
- value，v个字节，消息体，保存实际的消息数据

Kafka使用紧凑的二进制字节数组来保存消息，没有多余的比特位浪费。

# 使用场景

适合处理生产环境中流式数据：

- 消息传输
- 网站行为日志追踪
- 审计数据收集
- 日志收集
- Event Sourcing
- 流式处理
