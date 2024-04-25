---
title: '第1章:MySQL架构与历史'
date: 2020-12-15 09:28:13
tags: [db,mysql]
---



## _consumer_offsets剖析

## EOS（exactly once send）的实现

### 1. 幂等性

- 生产者引入了producer id(PID) 和 sequence number 2个概念
  - 分别对应RecordBatch的pid 和 first sequence 这两个字段
  - 每个生产者启动时，会被分配一个pid，并且维护了Map(<Pid, 分区>, sequence number)的关系；
    - 生产者发到partition的每条消息，都带有sequence number, 每发送一条，+1
- broker内存也维护了Map(<Pid, 分区>, sequence number)的关系
  - 如果生产者的sq  <  broker内存的sq + 1, 那么说明重复发送
  - 如果生产者的sq  > broker内存的sq + 1, 那么说明有消息丢失
    - 生产者会抛出异常OutOfOrderSeqExcepton



局限性：

- 只能保证单个生产者会话session中单分区的幂等性
- 并不能保障消息内容的幂等性。对于kafka来说，消息a即使内容一样，但是是2个不同的消息

```java
Record a = new Record;
producer.send(a);
producer.send(a);
```



### 2. 事务
