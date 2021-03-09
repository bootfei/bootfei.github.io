---
title: redis-6-锁与常见缓存问题
date: 2021-02-19 17:54:12
tags: [redis,lock]
---

# Redis实现乐观锁

## Redis乐观锁

乐观锁基于CAS(Compare And Swap)思想(比较并替换)，是不具有互斥性，不会产生锁等待而消 耗资源，但是需要反复的重试，但也是因为重试的机制，能比较快的响应。因此我们可以利用redis来 实现乐观锁。

具体思路如下:

- 监控 锁定量
- 如果该值被修改成功则表示该请求被通过， 反之表示该请求未通过。
- 从监控到修改到执行都需要在redis里操作，这样就需要用 到Redis事务。

```java
public void watch() {
    try {
        String watchKeys = "watchKeys"; 
        //初始值 
        value=1 jedis.set(watchKeys, 1); 
        //监听key为watchKeys的值 
      	jedis.watch(watchkeys);
        //开启事务
        Transaction tx = jedis.multi();
        //watchKeys自增加一 
      	tx.incr(watchKeys);
        //执行事务，如果其他线程对watchKeys中的value进行修改，则该事务将不会执行 
      	//通过redis事务以及watch命令实现乐观锁
        List<Object> exec = tx.exec();
        if (exec == null) {
        System.out.println("事务未执行"); } else {
        System.out.println("事务成功执行，watchKeys的value成功修改"); }
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        jedis.close();
} }
```



## Redis乐观锁实现秒杀

在生产环境里，经常会利用redis乐观锁来实现秒杀，Redis乐观锁是Redis事务的经典应用。

由于秒杀只有少部分请求能够成功，而大量的请求是并发产生的，所以如何确定哪个请求成功了，就是 由redis乐观锁来实现。

具体思路如下:

- 监控锁定量，如果该值被修改了，那么则会失败，反之，成功。[用户首先尝试获取秒杀的资格。(通过事务watch()函数)]()
- [用户获取秒杀资格后，再去检查库存是否有库存(通过事务检查库存)]()

```java
public class SecKill {
    public static void main(String[] arg) {
          //库存key
          String redisKey = "stock";
          ExecutorService executorService = Executors.newFixedThreadPool(20);
          try {
              Jedis jedis = new Jedis("127.0.0.1", 6378); 
              // 可以被秒杀的库存的初始值，库存总共20个 
              jedis.set(redisKey, "0");
              jedis.close();
          } catch (Exception e) {
            e.printStackTrace();
          }

        for (int i = 0; i < 1000; i++) {
          executorService.execute(() -> {
            Jedis jedis1 = new Jedis("127.0.0.1", 6378);
            try {
                jedis1.watch(redisKey);//乐观锁的核心业务
                String redisValue = jedis1.get(redisKey);
                int valInteger = Integer.valueOf(redisValue);
                String userInfo = UUID.randomUUID().toString();
                //用户成功获取抢夺的资格，然后检查库存量
                if (valInteger < 20) {
                    Transaction tx = jedis1.multi(); 
                  	tx.incr(redisKey);
                    List list = tx.exec();
                    // 秒成功 失败返回空list而不是空
                    if (list != null && list.size() > 0) {
                        System.out.println("用户:" + userInfo + "，秒杀成功!当前成功人数:" +(valInteger + 1));
                    }
                    // 版本变化，被别人抢了。 
                    else {
                        System.out.println("用户:" + userInfo + "，秒杀失败");
                    }
                }
                // 秒完了 
                else {
                    System.out.println("已经有20人秒杀成功，秒杀结束");
                }
            } catch (Exception e) {
              	e.printStackTrace();
            } finally {
              	jedis1.close();
            }
          }); 
        }

        executorService.shutdown();
    }
}
  
```



# Redis实现分布式锁

- 单应用中使用锁(单进程多线程)： synchronized、ReentrantLock 
- 分布式应用中使用锁(多进程多线程) ：分布式锁是控制分布式系统之间同步访问共享资源的一种方式

## 原理

利用Redis的单线程特性对共享资源进行串行化处理

## 获取锁

### 方式1（使用set命令实现）--推荐

- "NX":没有键值则设置

```java
/** * 使用redis的set命令实现获取分布式锁 
* @param lockKey 可以就是锁 
* @param requestId 请求ID，保证同一性 uuid+threadID 
* @param expireTime 过期时间，避免死锁 
* @return */
public boolean getLock(String lockKey,String requestId,int expireTime) { 
    //NX:保证互斥性 // hset 原子性操作 
    String result = jedis.set(lockKey, requestId, "NX", "EX", expireTime); 
    if("OK".equals(result)) { 
        return true; 
    }
    return false; 
}
```

### 方式2（使用setnx命令实现） -- 并发会产生问题

[因为2个redis操作，即设置key和设置expireTime，不是原子性的复合操作，所以会有并发问题]()

```java
public boolean getLock(String lockKey,String requestId,int expireTime) { 
    Long result = jedis.setnx(lockKey, requestId); 
    if(result == 1) {
        //成功设置 失效时间 
        jedis.expire(lockKey, expireTime); 
        return true; 
    }
    return false; 
}
```

## 释放锁

### 方式1（del命令实现） -- 并发会产生问题

```java
/*** 释放分布式锁 * @param lockKey * @param requestId */ 
public static void releaseLock(String lockKey,String requestId) { 
    if (requestId.equals(jedis.get(lockKey))) { 
        jedis.del(lockKey); 
    } 
}
```

> 问题在于如果调用jedis.del()方法的时候，这把锁已经不属于当前客户端的时候会解除他人加的 锁。
>
> 那么是否真的有这种场景？答案是肯定的，比如客户端A加锁，一段时间之后客户端A解锁，在执行 jedis.del()之前，锁突然过期了，此时客户端B尝试加锁成功，然后客户端A再执行del()方法，则将 客户端B的锁给解除了。

### 方式2（redis+lua脚本实现）-- 推荐

```java
public static boolean releaseLock(String lockKey, String requestId) { 
    String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
    Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId)); 
    if (result.equals(1L)) {
        return true; 
    }
    return false; 
}
```



# Redis常见缓存问题

## 数据读 

### 缓存穿透

> 查询缓存中永远不存在的一个值。因为这个值数据库没有。 

一般的缓存系统，都是按照key去缓存查询，如果不存在对应的value，就应该去后端系统查找（比如DB）。如果key对应的value是一定不存在的，并且对该key并发请求量很大，就会对后端系统造成很大的压力。

也就是说，对不存在的key进行高并发访问，导致数据库压力瞬间增大，这就叫做【缓存穿透】。

```
解决方案：

对查询结果为空的情况也进行缓存，缓存时间设置短一点，或者该key对应的数据insert了之后清理缓存。
```

### 缓存雪崩

> 大量缓存集中失效。 
>
> 大量数据访问压力一下怼到数据库身上 

当缓存服务器重启或者大量缓存集中在某一个时间段失效，这样在失效的时候，也会给后端系统(比如DB)带来很大压力。

突然间大量的key失效了或redis重启，大量访问数据库

```
解决方案:

1、 key的失效期分散开 不同的key设置不同的有效期

2、设置二级缓存

3、高可用
```

### 缓存穿透

> 单个缓存有效期到了，失效了。 
>
> 而且在这个时间点有大量的请求进来（导致这些请求都访问到数据库）。 

对于一些设置了过期时间的key，如果这些key可能会在某些时间点被超高并发地访问，是一种非常“热点”的数据。这个时候，需要考虑一个问题：缓存被“击穿”的问题，这个和缓存雪崩的区别在于这里针对某一key缓存，前者则是很多key。

缓存在某个时间点过期的时候，恰好在这个时间点对这个Key有大量的并发请求过来，这些请求发现缓存过期一般都会从后端DB加载数据并回设到缓存，这个时候大并发的请求可能会瞬间把后端DB压垮。

解决方案：

```
用分布式锁控制访问的线程

使用redis的setnx互斥锁先进行判断，这样其他线程就处于等待状态，保证不会有大并发操作去操作数据库。

if(redis.sexnx()==1){

 //先查询缓存

 //查询数据库

 //加入缓存

}

不设超时时间，写一致问题
```

## 数据写

> 数据不一致的根源 ： 数据源不一样

如何解决

```
强一致性很难，追求最终一致性
互联网业务数据处理的特点
高吞吐量
低延迟
数据敏感性低于金融业
时序控制是否可行？
先更新数据库再更新缓存或者先更新缓存再更新数据库
本质上不是一个原子操作，所以时序控制不可行
保证数据的最终一致性(延时双删)
1、先更新数据库同时删除缓存项(key)，等读的时候再填充缓存
2、2秒后再删除一次缓存项(key)
3、设置缓存过期时间 Expired Time 比如 10秒 或1小时
4、将缓存删除失败记录到日志中，利用脚本提取失败记录再次删除（缓存失效期过长 7*24）
```

升级方案

```
通过数据库的binlog来异步淘汰key，利用工具(canal)将binlog日志采集发送到MQ中，然后通过ACK机制确认处理删除缓存。
```

