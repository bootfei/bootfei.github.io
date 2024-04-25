---
title: '第1章:MySQL架构与历史'
date: 2020-12-15 09:28:13
tags: [db,mysql]
---



- 主题和分区是逻辑上的概念
- 分区可以有一到多个副本，每个副本对应一个日志文件，每个日志文件对应一到多个日志分段LogSegment，每个日志分段还可细分为索引文件、日志存储文件、快照文件。



![topic-partition](https://cxis.me/Kafka%e4%b8%ad%e7%9a%84%e4%b8%bb%e9%a2%98%e5%92%8c%e5%88%86%e5%8c%ba/topic-partition-1.png)



- 分区使用多副本机制提升可靠性，只有leader副本对外提供读写服务，follower副本只负责在内部进行消息同步。如果一个分区的leader副本不可用，Kafka会从剩余的follower副本中挑选一个新的leader副本来继续对我提供服务



```shell
创建1个topic =‘topic-create’，要求分区=4，副本因子=2
如果broker node = 3
#node 1: brokerId=0
	topic-create-0 : 主题是'topic-create'-分区-0 
	topic-create-1 :
#node 2: brokerId=1
	topic-create-1
	topic-create-2
	topic-create-3
#node 3: brokerId=2
	topic-create-0
	topic-create-2
	topic-create-3
```

