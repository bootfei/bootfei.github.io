---
title: 分布式锁-01-redis实现
date: 2021-05-20 11:42:40
tags:
---

# 实际应用

## 故障

```
java.lang.IllegalMonitorStateException: attempt to unlock lock, not locked by current thread by node id: def1bd2c-1f49-4802-b635-5ea78543c033 thread-id: 109
	at org.redisson.RedissonLock.unlock(RedissonLock.java:366)
```

## 排查原因

- 先检查项目中是否确保了 **redissonClient 的单例**。
  - 定位依据：我还没想到，因为用spring都是单例

- redissonClient 虽然是单例的静态成员变量，但初始化时未加锁，而是简单使用  （本次报错原因）

  - 定位依据：不需要改代码甚至debug，只需要搜索日志里是否有两行，打印两次版本信息说明肯定初始化了两次 Redisson。

    ```
    13:58:07.972 [main] INFO org.redisson.Version - Redisson 2.8.2
    ```


```
private static RedissonClient redisson = null;
      
public static RedissonClient getRedisson(){
  if(redisson == null){
      RedissonManager.init(); //初始化
  }
  return redisson;
}
```

- 加锁解锁没有同一个 lock，而是每次都使用`getRedisson().getLock(key)` 获取lock1和lock2两个不同的lock对象。导致解锁时从另一个 redissonClient 并没有获取到锁。换言之，报错里的`not locked by current thread by node id: def1bd2c-1f49-4802-b635-5ea78543c033 thread-id: 109` 其实关键问题在于`by node id` ，而不是线程id。
  - 定位依据：加锁和解锁时，打印lock对象，对象信息不一致，说明是两个对象。
- 解锁时，当前线程不是锁的持有者，关键问题在于`by current thread` ，而不是 node id。
  - 定位依据：使用lock.isHeldByCurrentThread()判断，如果不是被当前线程持有锁，当前线程释放释放锁肯定报这个异常



