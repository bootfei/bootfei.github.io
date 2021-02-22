---
title: redis-6-锁与常见缓存问题
date: 2021-02-19 17:54:12
tags: [redis,lock]
---

# Redis实现乐观锁：秒杀

具体参考[事务专题]()中的乐观锁

# Redis实现分布式锁

- 单应用中使用锁(单进程多线程)： synchronized、ReentrantLock 
- 分布式应用中使用锁(多进程多线程) ：分布式锁是控制分布式系统之间同步访问共享资源的一种方式