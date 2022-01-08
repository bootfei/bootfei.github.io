---
title: Redis-线上高延迟排查
date: 2021-05-06 12:04:04
tags:
---



### 一条命令执行过程

在本文场景下，延迟 (latency) 是指从客户端发送命令到客户端接收到命令返回值的时间间隔。

![Image](https://mmbiz.qpic.cn/mmbiz_png/9QbuglRCMtdhic9zTZh5xCllEzgLWRcOXQoemRPGek07hlpV6XpJsFTsnZuTyeVWg7Gymsd7yU84UOhwXMUSe3A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

上图是 Redis 客户端发送一条命令的执行过程示意图，绿色的是执行步骤，而蓝色的则是可能出现的导致高延迟的原因。

#### 网络连接限制、网络传输速率和CPU性能等是所有服务端都可能产生的性能问题

####  Redis 自己可能导致高延迟

- 命令或者数据结构误用、持久化阻塞和内存交换。
- 而且，Redis 采用[单线程]()和[事件驱动的机制]()来处理网络请求，分别有对应的[连接应答处理器]()，[命令请求处理器]()和[命令回复处理器]()来处理客户端的网络请求事件，处理完一个事件就继续处理队列中的下一个。一条命令处理出现了高延迟会影响接下来处于排队状态的其他命令。

<img src="https://mmbiz.qpic.cn/mmbiz_png/9QbuglRCMtdhic9zTZh5xCllEzgLWRcOXscLk1bRExg5BlLia5K6hRZQz1gUakMksxtLjkl7ibMDMz8m9ib2evyEYg/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom:50%;" />

### 问题排查

对于高延迟，Redis 原生提供慢查询统计功能，执行 slowlog get {n} 命令可以获取最近的 n 条慢查询命令，默认对于执行超过10毫秒(可配置)的命令都会记录到一个定长队列中，线上实例建议设置为1毫秒便于及时发现毫秒级以上的命令。

```
# 超过 slowlog-log-slower-than 阈值的命令都会被记录到慢查询队列中
# 队列最大长度为 slowlog-max-lenslowlog-log-slower-than 10000s lowlog-max-len 128
```

如果命令执行时间在毫秒级，则实例实际OPS只有1000左右。慢查询队列长度默认128，可适当调大。慢查询本身只记录了命令执行时间，不包括数据网络传输时间和命令排队时间，因此客户端发生阻塞异常 后，可能不是当前命令缓慢，而是在等待其他命令执行。需要重点比对异常和慢查询发生的时间点，确认是否有慢查询造成的命令阻塞排队。

#### 不合理的命令或者数据结构

比如对一个包含上万个元素的 hash 结构执行 hgetall 操作，由于数据量比较大且命令算法复杂度是 O(n)，这条命令执行速度必然很慢。

这个问题就是典型的不合理使用命令和数据结构。对于高并发的场景我们应该尽量避免在大对象上执行算法复杂度超过 O(n) 的命令。对于键值较多的 hash 结构可以使用 scan 系列命令来逐步遍历，而不是直接使用 hgetall 来全部获取。

Redis 本身提供发现大对象的工具，对应命令：redis-cli-h {ip} -p {port} bigkeys。这条命令会使用 scan 从指定的 Redis DB 中持续采样，实时输出当时得到的 value 占用空间最大的 key 值，并在最后给出各种数据结构的 biggest key 的总结报告。

```
> redis-cli -h host -p 12345 --bigkeys
```

#### 持久化阻塞

对于开启了持久化功能的Redis节点，需要排查是否是持久化导致的阻塞。持久化引起主线程阻塞的操作主要有：fork 阻塞、AOF刷盘阻塞。

fork 操作发生在 RDB 和 AOF 重写时，Redis 主线程调用 fork 操作产生共享内存的子进程，由子进程完成对应的持久化工作。如果 fork 操作本身耗时过长，必然会导致主线程的阻塞。

<img src="https://mmbiz.qpic.cn/mmbiz_png/9QbuglRCMtdhic9zTZh5xCllEzgLWRcOXA2v2rmicbJ08REVFIiaYfMbvnJtMTqCV27OtND6ZCkibB9ia69ZWlmr3rw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom:67%;" />

Redis 执行 fork 操作产生的子进程内存占用量表现为与父进程相同，理论上需要一倍的物理内存来完成相应的操作。但是 Linux 具有写时复制技术 (copy-on-write)，父子进程会共享相同的物理内存页，当父进程处理写请求时会对需要修改的页复制出一份副本完成写操作，而子进程依然读取 fork 时整个父进程的内存快照。所以，一般来说，fork 不会消耗过多时间。

可以执行 `info stats`命令获取到 latestforkusec 指标，表示 Redis 最近一次 fork 操作耗时，如果耗时很大，比如超过1秒，则需要做出优化调整。

```
> redis-cli -c -p 7000 info | grep -w latest_fork_useclatest_fork_usec:315
```

当我们开启AOF持久化功能时，文件刷盘的方式一般采用每秒一次，后台线程每秒对AOF文件做 fsync 操作。当硬盘压力过大时，fsync 操作需要等待，直到写入完成。如果主线程发现距离上一次的 fsync 成功超过2秒，为了数据安全性它会阻塞直到后台线程执行 fsync 操作完成。这种阻塞行为主要是硬盘压力引起，可以查看 Redis日志识别出这种情况，当发生这种阻塞行为时，会打印如下日志：

```
Asynchronous AOF fsync is taking too long (disk is busy). \Writing the AOF buffer without waiting for fsync to complete, \this may slow down Redis.
```

也可以查看 info persistence 统计中的 aofdelayedfsync 指标，每次发生 fdatasync 阻塞主线程时会累加。

```
>info persistenceloading:0aof_pending_bio_fsync:0aof_delayed_fsync:0
```

#### 内存交换

内存交换（swap）对于 Redis 来说是非常致命的，Redis 保证高性能的一个重要前提是所有的数据在内存中。如果操作系统把 Redis 使用的部分内存换出到硬盘，由于内存与硬盘读写速度差几个数量级，会导致发生交换后的 Redis 性能急剧下降。识别 Redis 内存交换的检查方法如下：

```
>redis-cli -p 6383 info server | grep process_id # 查询 redis 进程号
>cat /proc/4476/smaps | grep Swap # 查询内存交换大小Swap: 0 kBSwap: 4 kBSwap: 0 kBSwap: 0 kB
```

如果交换量都是0KB或者个别的是4KB，则是正常现象，说明Redis进程内存没有被交换。

有很多方法可以避免内存交换的发生。比如说：

- 保证机器充足的可用内存
- 确保所有Redis实例设置最大可用内存(maxmemory)，防止极端情况下 Redis 内存不可控的增长。
- 降低系统使用swap优先级，如 `echo 10>/proc/sys/vm/swappiness`。