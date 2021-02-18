---
title: redis-5-主从与哨兵与集群
date: 2021-01-14 15:59:03
tags: [redis]
---

# 主从复制

- 主 Redis 中的数据有两个副本( replication )即从 redis1 和从 redis2 ，即使一台 Redis 服 务器宕机其它两台 Redis 服务也可以继续提供服务。
- 主 Redis 中的数据和从 Redis 上的数据保持实时同步，当主 Redis 写入数据时通过主从复制机制 会复制到两个从 Redis 服务上。
- 只有一个主 Redis ，可以有多个从 Redis 。 主从复制不会阻塞master，在同步数据时，master 可以继续处理client 请求。 一个 Redis 可以即是主又是从，如下图
  <img src="https://images2017.cnblogs.com/blog/1227483/201802/1227483-20180201103511703-1604168118.png" alt="img" style="zoom:50%;" />

## 主配置

无

## 从配置

修改从服务器上的 redis.conf 文件:

```shell
# slaveof <masterip> <masterport>
# 表示当前【从服务器】对应的【主服务器】的IP是192.168.10.135，端口是6379。 
slaveof 192.168.10.135 6379  #老版本命令
replicaof 192.168.19.135 6379 #新版本
```

## 实现原理

<img src="https://pic4.zhimg.com/80/v2-cc94828d6884eac1cc59ea312727307b_1440w.jpg" alt="img" style="zoom:67%;" />

- Redis 的主从同步，分为[全量同步]() 和[增量同步]() 
- 只有从机第一次连接上主机是全量同步 。
- 断线重连有可能触发[全量同步]()也有可能是[增量同步]() ( master 判断 runid 是否一致)。
- 除此之外的情况都是[增量同步]() 。

### 全量同步

<img src="https://pic3.zhimg.com/80/v2-40e7dd00ad78ec9f3fd8c1cb36e32b26_1440w.jpg" alt="img" style="zoom:50%;" />

- 同步快照阶段：Master 创建并发送快照给 Slave ， Slave 载入并解析快照。 Master 同时将此阶段所产生的新的写命令存储到缓冲区。
- 同步写缓冲阶段：Master 向 Slave 同步存储在缓冲区的写操作命令。 <!--解决第一步中产生的问题-->
- 同步增量阶段：Master 向 Slave 同步写操作命令。<!--第二部是增量操作-->

### 增量同步



- Redis 增量同步主要指 **Slave 完成初始化后开始正常工作**时，**Master 发生的写操作同步到 Slave 的过程**。
- 通常情况下，Master 每执行一个写命令就会向 Slave 发送相同的**写命令**，然后 Slave接收并执行。