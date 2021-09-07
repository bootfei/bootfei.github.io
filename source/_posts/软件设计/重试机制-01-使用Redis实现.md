---
title: 重试机制-01-使用Redis实现
date: 2021-09-02 13:54:44
tags:
---

https://mp.weixin.qq.com/s/rBJYGiV4Jq8qhSb4Fcmb9A

## 使用场景

- 超时重试，一方面保障业务成功，另一方面保障重试操作不会积压



## Redis实现方式

### 方案

- 将**回调任务**丢入一个**回调队列**中，由指定的线程去处理队列中的**回调任务**。

- 为了避免本地内存的丢失，所以队列可以使用**高可用**的redis集群。
- 就绪队列

![Image](https://mmbiz.qpic.cn/mmbiz_png/48MFTQpxichlDnH6mldQ6LZseXgnLkolmjWs1XAFicqE1Wtc8cLMsIxeuUM6xiau1wCLuCJn8tU4MWLrNN9OFhO4w/640)

![Image](https://mmbiz.qpic.cn/mmbiz_png/48MFTQpxichlDnH6mldQ6LZseXgnLkolm57lXIGDCsusx8eUDCV6UTkCScChtpTAiaGuHgB96dK6GVfooqPrTNoA/640)

### 设计锚点

#### **回调任务失败如何处理，直接重新丢入回调队列吗？**

一般出现回调失败的原因大部分都是因为系统存在一定的bug因素导致的，需要人工介入手工修复的机率较大，所以回调的间隔一般不会是连续性的立即回调，而是呈现一个阶段性的回调处理，例如分别间隔3秒，30秒，1min，3min，9min，30min，60min...的间隔进行回调尝试，如果经过多次回调都始终处于失败状态，则需要进行专门的记录。

 

 

#### **回调队列的任务获取到之后，什么时候执行任务？**

这里可以利用到一个redis的zset数据结构。zset结构中基本存储信息和set结构类似，但是会给每个元素都有一个排分的score标示，根据排分的大小会在内存中进行排序操作。所以可以将时间戳作为排分的计数项，时间戳靠前的排在zset的前头，那么在进行回调任务处理的时候就会被优先读取出来。

从回调队列取出的任务，判断当前的时间戳是否小于或等于预期的执行时间：是，进行业务回调了。否，？

 

#### **回调队列的任务，什么时候删除？**

**回调任务**必须在**回调成功**之后再去进行删除。但是如果回调过程中如果网络通信较慢，可能会导致回调队列的消费效率较低，容易出现堵塞的情况发生。

所以这里引入了一个**就绪队列**的设计  <!--这里过度设计了，因为实际情况中，回调队列的长度是正常业务的千分之一-->

当回调队列中的任务的执行时间达到预期的时候，就将其放入到就绪队列中，这份转移的工作交给了专门的就绪线程去处理。并且一旦转移到就绪队列之后，便将其从回调队列移除。

就绪队列是采用了redis的queue队列进行存储，满足了先进先出的优先级设计需求。



### 代码设计

#### 回调任务

```java
public class DelayTaskInfo implements Serializable {
    private static final long serialVersionUID = -1003963035927679825L;
    private long taskId;
    private String topic;
    private Long executeTime;
    private Long executeTimeout;
    private int retryTimes;
    private String message;
}
```

```java
public class DelayTaskSortItem implements Serializable {
    private static final long serialVersionUID = -2003963035927679825L;
    private long taskId;
    private String topic;
    private Long executeTime;
    private Long executeTimeout;
    private int retryTimes;
    private String message;
}
```



#### 回调队列

```java
public interface RedisQueue<T> {
    /**
     * 弹出任务
     */
    T pop();

    /**
     * 放入任务
     */
    boolean put(T t);

    /**
     * 删除队列中的一项内容
     */
    boolean delete(T item);
}
```

```java
@Component("redisDelayQueue")
@Slf4j
public class RedisDelayQueue implements RedisQueue<DelayTaskSortItem> {
    @Resource
    private RedisService redisService;
    private static String redisDelayQueueKey = "redis:delay:pay:notify";
    private static final Object mutex = new Object();
  
    @Override
    public DelayTaskSortItem pop() {
        synchronized (mutex) {
            Set<Tuple> rangeSet = redisService.zRangeWithScores(redisDelayQueueKey, 0, 1);
            if (CollectionUtils.isEmpty(rangeSet)) {
                return null;
            }
            for (Tuple tuple : rangeSet) {
                long currentTimeMillis = System.currentTimeMillis();
                double expireTime = tuple.getScore();
                if ((currentTimeMillis - expireTime) > 0) {
                    DelayTaskSortItem delayTaskInfo = JSON.parseObject(tuple.getElement(), DelayTaskSortItem.class);
                    return delayTaskInfo;
                }
            }
        }
        return null;
    }
  
    @Override
    public boolean put(DelayTaskSortItem delayTaskSortItem) {
        redisService.zAdd(redisDelayQueueKey, delayTaskSortItem.getExecuteTime(), JSON.toJSONString(delayTaskSortItem));
        return true;
    }
  
    @Override
    public boolean delete(DelayTaskSortItem delayTaskSortItem) {
        redisService.zRem(redisDelayQueueKey, Arrays.asList(String.valueOf(JSON.toJSONString(delayTaskSortItem))));
        return true;
    }
}
```



#### 回调线程

将回调队列中的元素转换到就绪队列的专门处理线程：

```java
@Component
@Slf4j
public class RedisDelayQueueHandler implements Runnable {
    @Resource(name = "redisDelayQueue")
    private RedisQueue<DelayTaskSortItem> redisDelayQueue;
    @Resource(name = "redisReadyQueue")
    private RedisQueue<DelayTaskInfo> redisReadyQueue;
    @Resource
    private IDistributionLock distributionLock;
   
  /*@SneakyThrows - Project Lombok
@SneakyThrows can be used to sneakily throw checked exceptions without actually declaring this in your method's throws clause.
  */
    @SneakyThrows
    @Override
    public void run() {
        while (true) {
            try {
                DelayTaskSortItem delayTaskSortItem= redisDelayQueue.pop();
                if (delayTaskSortItem == null) {
                    continue;
                }
                String lockKey = "pay:notify:dequeue:lock" + delayTaskSortItem.getTaskId();
                try {
                    boolean lockStatus = distributionLock.tryLockTimeOut(lockKey, 3000);
                    if (!lockStatus) {
                        continue;
                    }
                    if(delayTaskSortItem.getExecuteTime() > System.currentTimeMillis()){
                        continue;
                    }
 
                    if(delayTaskInfo.getExecuteTime() > System.currentTimeMillis()) {
                        continue;
                    }
                    //推入任务到达就绪队列之前需要检查是否符合条件：执行任务时间，任务池和延迟队列任务是否一致，任务id是否一致
                    log.info("推入任务："+delayTaskInfo);
                    redisReadyQueue.put(delayTaskInfo);
                    redisDelayQueue.delete(delayTaskSortItem);
                   
                }catch (Exception e){
                    log.info("支付的延迟队列出现异常情况1：", e);
                }finally {
                    distributionLock.releaseLock(lockKey);
                }
            } catch (Exception e) {
                log.info("支付的延迟队列出现异常情况2：", e);
            }
        }
    }
}
```



#### 就绪队列

```java
@Component("redisReadyQueue")
public class RedisReadyQueue implements RedisQueue<DelayTaskInfo> {
    @Resource
    private RedisService redisService;

    private static final String REDIS_READY_QUEUE_KEY = "redis:ready:queue:pay";

    @Override
    public DelayTaskInfo pop() {
        String item = redisService.rpop(REDIS_READY_QUEUE_KEY);
        if(item != null) {
            return JSON.parseObject(item,DelayTaskInfo.class);
        }
        return null;
    }

    @Override
    public boolean put(DelayTaskInfo delayTaskInfo) {
        Long id = -1L;
        id = redisService.lpush(REDIS_READY_QUEUE_KEY, JSON.toJSONString(delayTaskInfo));
        return id > 0;
    }

    @Override
    public boolean delete(DelayTaskInfo delayTaskInfo) {
        return true;
    }    
}
```



#### 就绪线程

```java
@Component
@Slf4j
public class RedisReadyQueueHandler implements Runnable {
    @Resource
    private RedisReadyQueue redisReadyQueue;
    @Resource
    private RedisDelayQueue redisDelayQueue;

    @SneakyThrows
    @Override
    public void run() {
        while (true) {
            DelayTaskInfo delayTaskInfo = redisReadyQueue.pop();
            if (delayTaskInfo == null) {
                continue;
            }
            log.info("弹出任务：delayTaskInfo is {}",delayTaskInfo);
            ResponseEntity<String> responseEntity = null;
            try {
                if (HttpStatus.OK.equals(responseEntity.getStatusCode())) {
                    log.info("支付回调成功！====== ");
                } else {
                    throw new RuntimeException("支付业务回调失败");
                }
            }catch (Exception e){
                retryAgain(delayTaskInfo);
            }
        }
    }
   
    /**
     * 进入重试
     */
    public void retryAgain(DelayTaskInfo delayTaskInfo){
        //进入支付重试环节
        delayTaskInfo.setExecuteTime(System.currentTimeMillis() + RETRY_TIMES[retryTime] * 1000 );
        delayTaskInfo.setExecuteTimeout(delayTaskInfo.getExecuteTimeout() + DEFAULT_TIME_OUT);
        delayTaskInfo.setRetryTimes(retryTime+1);
      
        DelayTaskSortItem delayTaskSortItem = new DelayTaskSortItem(delayTaskInfo.getTaskId(),PAY_TOPIC,delayTaskInfo.getExecuteTime());
      
        redisDelayQueue.put(delayTaskSortItem);
    }
}
```

