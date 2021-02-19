---
title: redis-4-主从与哨兵与集群
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



# 哨兵机制

Redis 主从复制的缺点:没有办法对 master 进行动态选举，需要使用 Sentinel 机制完成动态选举

## 简介

Redis 哨兵（Sentinel）是 Redis 的**高可用性**（Hight Availability）解决方案：由一个或多个 Sentinel 实例组成的 Sentinel 系统可以监视任意多个主服务器，以及这些主服务器的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求。

<img src="http://dunwu.test.upcdn.net/snap/20200131135847.png" alt="img" style="zoom:33%;" />



## Sentinel主要功能

- **`监控（Monitoring）`** - Sentinel 不断检查主从服务器是否正常在工作。
- **`通知（Notification）`** - Sentinel 可以通过一个 api 来通知系统管理员或者另外的应用程序，被监控的 Redis 实例有一些问题。
- **`自动故障转移（Automatic Failover）`** - 如果一个主服务器下线，Sentinel 会开始自动故障转移：把一个从节点提升为主节点，并重新配置其他的从节点使用新的主节点，使用 Redis 服务的应用程序在连接的时候也被通知新的地址。
- **`配置提供者（Configuration provider）`** - Sentinel 给客户端的服务发现提供来源：对于一个给定的服务，客户端连接到 Sentinels 来寻找当前主节点的地址。当故障转移发生的时候，Sentinel 将报告新的地址。

## 启动Sentinel

```shell
redis-sentinel /path/to/sentinel.conf
redis-server /path/to/sentinel.conf --sentinel
```

**Sentinel 本质上是一个运行在特殊状模式下的 Redis 服务器**。

## 故障判定原理

- 通过ping-pang机制使用Sentinel的监控功能：
  1. 默认情况下，**每个** `Sentinel` 节点会以 **每秒一次** 的频率对 `Redis` 节点和 **其它** 的 `Sentinel` 节点发送 `PING` 命令，并通过节点的 **回复** 来判断节点是否在线。
     - **主观下线ODOWN**：**主观下线** 适用于所有 **主节点** 和 **从节点**。如果在 `down-after-milliseconds` 毫秒内，`Sentinel` 没有收到 **目标节点** 的有效回复，则会判定 **该节点** 为 **主观下线**。
     - **客观下线SDOWN**：**客观下线** 只适用于 **主节点**。当 `Sentinel` 将一个主服务器判断为主管下线后，为了确认这个主服务器是否真的下线，会向同样监视这一主服务器的其他 Sentinel 询问，看它们是否也认为主服务器已经下线。当足够数量的 Sentinel 认为主服务器已下线，就判定其为客观下线，并对其执行故障转移操作。
  2. 如果一个实例(instance)距离最后一次有效回复 PING 命令的时间超过 down-after- milliseconds 选项所指定的值， 则这个实例会被 Sentinel(哨兵)进程标记为(ODOWN)。
  3. 如果一个Master主服务器被标记为主观下线(SDOWN)，则正在监视这个Master主服务器的所有 Sentinel(哨兵)以每秒一次的频率确认Master主机的确进入SDOWN
  4. 当足够数量的 Sentinel (>=配置文件中的值)认为主服务器已下线，就判定其为客观下线ODWON
- 获取服务器信息：
  5. 在一般情况下， 每个 Sentinel(哨兵)进程会以每 10 秒一次的频率向集群中的所有Master主服务器、Slave从服务器发送 INFO 命令。**Sentinel 向主服务器发送 `INFO` 命令，获取主服务器及它的从服务器信息**。
  6. 当Master主服务器被 Sentinel(哨兵)进程标记为ODOWN时，Sentinel(哨兵)进程向[下线的 Master主服务器]()的[所有 Slave从服务器]()发送 INFO 命令的频率会从10秒一次改为每秒一次。
- 故障判断：
  7. 若没有足够数量的 Sentinel(哨兵)进程同意 Master主服务器下线， Master主服务器的客观下线状态就会被移除。若 Master主服务器重新向 Sentinel(哨兵)进程发送 PING 命令 返回有效回复， Master 主服务器的主观下线状态就会被移除。

## 自动故障迁移

- 它会将失效Master 的其中一个Slave升级为新的Master , 并让失效Master的其他Slave改为复制新的 Master ;
- 当客户端试图连接失效的 Master 时，集群也会向客户端返回新 Master 的地址，使得集群可以使 用现在的Master 替换失效 Master 。
- Master 和 Slave 服务器切换后， [Master 的 redis.conf]() 、 [Slave 的 redis.conf]() 和[sentinel.conf]() 的配置文件的内容都会发生相应的改变，即Master 主服务器的 redis.conf 配置文件中会多一行 slaveof 的配置， sentinel.conf 的监控目标会随之调换。

# Redis集群

**[Redis 集群（Redis Cluster） (opens new window)](https://redis.io/topics/cluster-tutorial)是 Redis 官方提供的分布式数据库方案**。

既然是分布式，自然具备分布式系统的基本特性：可扩展、高可用、一致性。

- Redis 集群通过划分 hash 槽来分片，进行数据分享。
- Redis 集群采用主从模型，提供复制和故障转移功能，来保证 Redis 集群的高可用。
- 根据 CAP 理论，Consistency、Availability、Partition tolerance 三者不可兼得，而 Redis 集群的选择是 AP。Redis 集群节点间采用异步通信方式，不保证强一致性，尽力达到最终一致性。

## 架构细节

(1)所有的redis节点彼此互联(ping-pong机制 ),内部使用二进制协议优化传输速度和带宽. 

(2)节点的fail是通过集群中超过半数的节点检测失效时才生效.

(3)客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一 个可用节点即可

(4)redis-cluster把所有的物理节点映射到[0-16383] 上,cluster 负责维护node\slot\value

## Redis Cluster 分区

### 集群节点

Redis 集群由多个节点组成，节点刚启动时，彼此是相互独立的。**节点通过握手（ `CLUSTER MEET` 命令）来将其他节点添加到自己所处的集群中**。

向一个节点发送 `CLUSTER MEET` 命令，可以让当前节点与指定 IP、PORT 的节点进行握手，握手成功时，当前节点会将指定节点加入所在集群。

**集群节点保存键值对以及过期时间的方式与单机 Redis 服务完全相同**。

Redis 集群节点分为主节点（master）和从节点（slave），其中[主节点用于处理槽]()，而从节点则用于复制某个主节点，并在被复制的主节点下线时，代替下线主节点继续处理命令请求。

### 分配 Hash 槽

分布式存储需要解决的首要问题是把 **整个数据集** 按照 **分区规则** 映射到 **多个节点** 的问题，即把 **数据集** 划分到 **多个节点** 上，每个节点负责 **整体数据** 的一个 **子集**。

**Redis 集群通过划分 hash 槽来将数据分区**。Redis 集群通过分片的方式来保存数据库的键值对：**集群的整个数据库被分为 16384 个哈希槽（slot）**，数据库中的每个键都属于这 16384 个槽的其中一个，集群中的每个节点可以处理 0 个或最多 16384 个槽。**如果数据库中有任何一个槽没有得到处理，那么集群处于下线状态**。

通过向节点发送 [`CLUSTER ADDSLOTS` (opens new window)](https://redis.io/commands/cluster-addslots)命令，可以将一个或多个槽指派给节点负责。

```text
> CLUSTER ADDSLOTS 1 2 3
OK
```

集群中的每个节点负责一部分哈希槽，比如集群中有３个节点，则：

- 节点Ａ存储的哈希槽范围是：0 – 5500
- 节点Ｂ存储的哈希槽范围是：5501 – 11000
- 节点Ｃ存储的哈希槽范围是：11001 – 16384

### 寻址

当客户端向节点发送与数据库键有关的命令时，接受命令的节点会**计算出命令要处理的数据库属于哪个槽**，并**检查这个槽是否指派给了自己**：

- 如果键所在的槽正好指派给了当前节点，那么当前节点直接执行命令。
- 如果键所在的槽没有指派给当前节点，那么节点会向客户端返回一个 MOVED 错误，指引客户端重定向至正确的节点。

#### 计算键属于哪个槽

决定一个 key 应该分配到那个槽的算法是：**计算该 key 的 CRC16 结果再模 16834（2^14）**。

```text
slot = CRC16(KEY) & 16384
```

当节点计算出 key 所属的槽为 i 之后，节点会根据以下条件判断槽是否由自己负责：

```text
clusterState.slots[i] == clusterState.myself
```

#### MOVED 错误

当节点发现键所在的槽并非自己负责处理的时候，节点就会向客户端返回一个 `MOVED` 错误，指引客户端转向正在负责槽的节点。

`MOVED` 错误的格式为：

```text
MOVED <slot> <ip>:<port>
```

> 个人理解：MOVED 这种操作有点类似 HTTP 协议中的重定向。

### 重新分片

Redis 集群的**重新分片操作可以将任意数量的已经指派给某个节点（源节点）的槽改为指派给另一个节点（目标节点），并且相关槽所属的键值对也会从源节点被移动到目标节点**。

重新分片操作**可以在线进**行，在重新分片的过程中，集群不需要下线，并且源节点和目标节点都可以继续处理命令请求。

Redis 集群的重新分片操作由 Redis 集群管理软件 **redis-trib** 负责执行的，redis-trib 通过向源节点和目标节点发送命令来进行重新分片操作。

重新分片的实现原理如下图所示：

<img src="http://dunwu.test.upcdn.net/cs/database/redis/redis-cluster-trib.png" alt="img" style="zoom: 50%;" />

### ASK 错误

`ASK` 错误与 `MOVED` 的区别在于：**ASK 错误只是两个节点在迁移槽的过程中使用的一种临时措施**，在客户端收到关于槽 i 的 ASK 错误之后，客户端只会在接下来的一次命令请求中将关于槽 i 的命令请求发送至 ASK 错误所指示的节点，但这种转向不会对客户端今后发送关于槽 i 的命令请求产生任何影响，客户端仍然会将关于槽 i 的命令请求发送至目前负责处理槽 i 的节点，除非 ASK 错误再次出现。

判断 ASK 错误的过程如下图所示：

<img src="http://dunwu.test.upcdn.net/cs/database/redis/redis-ask.png" alt="img" style="zoom:50%;" />



## Redis Cluster 故障转移

### [#](https://dunwu.github.io/db-tutorial/nosql/redis/redis-cluster.html#复制)复制

Redis 复制机制可以参考：[Redis 复制](https://dunwu.github.io/db-tutorial/nosql/redis/redis-replication.html)

### [#](https://dunwu.github.io/db-tutorial/nosql/redis/redis-cluster.html#故障检测)故障检测

> 什么时候整个集群不可用(cluster_state:fail)?
>
> - 如果集群任意master挂掉,且当前master没有slave，则集群进入fail状态。也可以理解成集群的[0-16383]slot映射不完全时进入fail状态。 
>
> - 如果集群超过半数以上master挂掉，无论是否有slave，集群进入fail状态

**集群中每个节点都会定期向集群中的其他节点发送 PING 消息，以此来检测对方是否在线**。

节点的状态信息可以分为：

- 在线状态；
- 下线状态（FAIL）;
- 疑似下线状态（PFAIL），即在规定的时间内，没有应答 PING 消息；

### [#](https://dunwu.github.io/db-tutorial/nosql/redis/redis-cluster.html#故障转移)故障转移

1. 下线主节点的所有从节点中，会有一个从节点被选中。
2. 被选中的从节点会执行 `SLAVEOF no one` 命令，成为新的主节点。
3. 新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己。
4. 新的主节点向集群广播一条 PONG 消息，告知其他节点这个从节点已变成主节点。

#### [#](https://dunwu.github.io/db-tutorial/nosql/redis/redis-cluster.html#选举新的主节点)选举新的主节点

Redis 集群选举新的主节点流程基于[共识算法：Raft(opens new window)](https://www.jianshu.com/p/8e4bbe7e276c)

