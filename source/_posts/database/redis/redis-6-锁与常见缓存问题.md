---
title: redis-6-锁与常见缓存问题
date: 2021-02-19 17:54:12
tags: [redis,lock]
---

# 乐观锁

乐观锁基于CAS(Compare And Swap)思想(比较并替换)，是不具有互斥性，不会产生锁等待而消 耗资源，但是需要反复的重试，但也是因为重试的机制，能比较快的响应。因此我们可以利用redis来实 现乐观锁。具体思路如下:

- 利用redis的watch功能，监控这个redisKey的状态值 
- 获取redisKey的值
- 创建redis事务

- 在事务中给这个key的值+1，表示当前客户端在当前事务下正在对key操作
- 然后去执行这个事务，如果key的值被修改过则回滚，key不加1

> 最重要的实现就是watch命令，watch能够实现对某个变量的CAS中的Compare，在秒杀场景中就是要watch库存这个变量。

# 分布式锁

- 单应用中使用锁(单进程多线程)： synchronized、ReentrantLock 
- 分布式应用中使用锁(多进程多线程) ：分布式锁是控制分布式系统之间同步访问共享资源的一种方式