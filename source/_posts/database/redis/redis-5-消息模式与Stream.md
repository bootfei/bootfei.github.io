---
title: redis-5-主从复制与集群
date: 2021-01-14 15:59:03
tags: [redis,cluster]
---

# **Redis** **消息模式(了解，现在用**Stream)

## **队列模式** 

使用**list**类型的**lpush和rpop**实现消息队列

**注意事项:**

消息接收方如果不知道队列中是否有消息，会一直发送rpop命令，如果这样的话，会每一次都建 立一次连接，这样显然不好。 可以使用**brpop**命令，它如果从队列中取不出来数据，会一直阻塞，在一定范围内没有取出则返回 null

## **发布订阅模式**

略



# [Redis Stream(重点)](https://www.runoob.com/redis/redis-stream.html)

Redis 5.0 全新的数据类型:streams，官方把它定义为:以更抽象的方式建模日志的数据结构。Redis 的streams主要是一个append only的数据结构，至少在概念上它是一种在内存中表示的抽象数据类 型，只不过它们实现了更强大的操作，以克服日志文件本身的限制。

> 如果你了解MQ，那么可以把streams当做基于内存的MQ。如果你还了解kafka，那么甚至可以把 streams当做基于内存的kafka。

## 与订阅模式比较

另外，这个功能有点类似于redis以前的Pub/Sub，但是也有基本的不同:

- streams支持多个客户端(消费者)等待数据(Linux环境开多个窗口执行XREAD即可模拟)，并 且每个客户端得到的是完全相同的数据。 
- Pub/Sub是发送忘记的方式，并且不存储任何数据;而streams模式下，所有消息被无限期追加在 streams中，除非用于显示执行删除(XDEL)。

- streams的Consumer Groups也是Pub/Sub无法实现的控制方式。 

## streams数据结构

<img src="https://www.runoob.com/wp-content/uploads/2020/09/en-us_image_0167982791.png" alt="img" style="zoom:50%;" />

它主要有消息、生产者、消费者、消费组4组成

streams数据结构本身非常简单，但是streams依然是Redis到目前为止最复杂的类型，其原因是实现的 一些额外的功能:一系列的阻塞操作允许消费者等待生产者加入到streams的新数据。另外还有一个称 为Consumer Groups的概念，Consumer Group概念最先由kafka提出，Redis有一个类似实现，和 kafka的Consumer Groups的目的是一样的:[允许一组客户端协调消费相同的信息流]()!



## 案例

[客户端1：发布数据]()

```shell
#发布的第2条记录
127.0.0.1:6379> xadd mystream * message apple
"1589994652300-0"
127.0.0.1:6379> xadd mystream * message orange
"1589994679942-0"
```

[客户端2：读数据]()

```shell
127.0.0.1:6379> xrange mystream - +
1) 1) "1589994652300-0"
   2) 1) "message"
      2) "apple"
2) 1) "1589994679942-0"
   2) 1) "message"
   		2) "orange"
```

也可以阻塞读，这样必须等待客户端1发布数据

```
xread block 0 streams mystream $
```

现在[客户端1]()发布数据

```shell
#发布的第3条记录
127.0.0.1:6379> xadd mystream * message strawberry  
```

[客户端3:]()

创建消费者组

```shell
127.0.0.1:6379> xgroup create mustream mygroup1 0
OK
127.0.0.1:6379> xgroup create mustream mygroup2 0
OK
```

通过消费组读取消息

```shell
#消费者组zhange消费了前2个数据
127.0.0.1:6379> xreadgroup group mugroup1 zhange count 2 streams mystream >
1) 1) "mystream"
   2) 1) 1) "1589994652300-0"
         2) 1) "message"
            2) "apple"
      2) 1) "1589994679942-0"
         2) 1) "message"
            2) "orange"
#消费者组tuge消费了前2个数据
127.0.0.1:6379> xreadgroup group mugroup1 tuge count 2 streams mystream >
1) 1) "mystream"
   2) 1) 1) "1589995171242-0"
         2) 1) "message"
            2) "strawberry"
#因为消费者组tuge消费了前2个数据，现在只能消费第3个数据     
127.0.0.1:6379> xreadgroup group mugroup2 tuge count 1 streams mystream >
1) 1) "mystream"
   2) 1) 1) "1589995171242-0"
         2) 1) "message"
						2) "apple"
```



# **Redis Pipeline**（重要的优化技术）

Pipeline (管道技术) 是客户端提供的一种批处理技术 可以批量执行一组指令，一次性返回全部结果，

可以减少频繁的请求应答

- 使用pipeline技术

<img src="https://img-blog.csdnimg.cn/20181122105251930.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxbGd5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />

- 未使用pipeline技术
  <img src="https://img-blog.csdnimg.cn/20181122105343203.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3cxbGd5,size_16,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:50%;" />



# [Redis LUA脚本](https://www.runoob.com/redis/redis-scripting.html)（很简单）

常用于秒杀系统

