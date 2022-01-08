---
title: JDK-juc-CAS与自旋锁
date: 2021-04-01 09:21:03
tags:
---



# C A S基本概念

`C A S（compareAndSwap）`也叫比较交换，是一种无锁原子算法，映射到操作系统就是一条`cmpxchg`硬件汇编指令（**保证原子性**），其作用是让`C P U`将内存值更新为新值，但是有个条件，内存值必须与期望值相同，并且`C A S`操作无需用户态与内核态切换，直接在用户态对内存进行读写操作（**意味着不会阻塞/线程上下文切换**）。

它包含`3`个参数`C A S（V，E，N）`，`V`表示待更新的内存值，`E`表示预期值，`N`表示新值，当 `V`值等于`E`值时，才会将`V`值更新成`N`值，如果`V`值和`E`值不等，不做更新，这就是一次`C A S`的操作。

<img src="https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8ny1VNcSscicjAax5qNibFxqiabHujbJoOdS8SRxegxFTHuoKHLwjkVahNbPibZP6rhYkYtuX8RQkK2Cbw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom:50%;" />

简单说，`C A S`需要你额外给出一个期望值，也就是你认为这个变量现在应该是什么样子的，如果变量不是你想象的那样，说明它已经被别人修改过了，你只需要重新读取，设置新期望值，再次尝试修改就好了。

# C A S如何保证原子性

原子性是指一个或者多个操作在`C P U`执行的过程中不被中断的特性，要么执行，要不执行，不能执行到一半（**不可被中断的一个或一系列操作**）。

为了保证`C A S`的原子性，`C P U`提供了下面两种方式

- **总线锁定**
- **缓存锁定**

## 总线锁定

总线（`B U S`）是计算机组件间的传输数据方式，也就是说`C P U`与其他组件连接传输数据，就是靠总线完成的，比如`C P U`对内存的读写。

![Image](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8ny1VNcSscicjAax5qNibFxqiab1UEcNV0VfpBkR7icdiazr27wBlcCa8QaMvdHoDWuZTe3HqaLHAiaB3cXQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**总线锁定**是指`C P U`使用了总线锁，所谓总线锁就是使用`C P U`提供的`LOCK#`信号，当`C P U`在总线上输出`LOCK#`信号时，其他`C P U`的总线请求将被阻塞。

![Image](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8ny1VNcSscicjAax5qNibFxqiab183RLQ5wiaWzgiaiblQpR7JQP0HHdZxzHlXJgcrhB5VZXSFqBnYNblfsg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 缓存锁定

总线锁定方式虽然保证了原子性，但是在锁定期间，会导致大量阻塞，增加系统的性能开销，所以现代`C P U`为了提升性能，通过锁定范围缩小的思想设计出了缓存行锁定（**缓存行是`C P U`高速缓存存储的最小单位**）。

所谓**缓存锁定**是指`C P U`对**缓存行**进行锁定，当缓存行中的共享变量回写到内存时，其他`C P U`会通过总线嗅探机制感知该共享变量是否发生变化，如果发生变化，让自己对应的共享变量缓存行失效，重新从内存读取最新的数据，缓存锁定是基于缓存一致性机制来实现的，因为缓存一致性机制会阻止两个以上`C P U`同时修改同一个共享变量（**现代`C P U`基本都支持和使用缓存锁定机制**）。

# C A S的问题

`C A S`和锁都解决了原子性问题，和锁相比没有阻塞、线程上下文你切换、死锁，所以`C A S`要比锁拥有更优越的性能，但是`C A S`同样存在缺点。

`C A S`的问题如下

- **只能保证一个共享变量的原子操作**
- **自旋时间太长（建立在自旋锁的基础上）**
- **`ABA`问题**

## 只能保证一个共享变量原子操作

`C A S`只能针对一个共享变量使用，如果多个共享变量就只能使用锁了，当然如果你有办法把多个变量整成一个变量，利用`C A S`也不错，例如读写锁中`state`的高低位。

## ABA问题

`C A S`需要检查待更新的内存值有没有被修改，如果没有则更新，但是存在这样一种情况，如果一个值原来是`A`，变成了`B`，然后又变成了`A`，在`C A S`检查的时候会发现没有被修改。

假设有两个线程，线程`1`读取到内存值`A`，线程`1`时间片用完，切换到线程`2`，线程`2`也读取到了内存值`A`，并把它修改为`B`值，然后再把`B`值还原到`A`值，简单说，修改次序是`A->B->A`，接着线程`1`恢复运行，它发现内存值还是`A`，然后执行`C A S`操作，这就是著名的`ABA`问题，但是好像又看不出什么问题。

只是简单的数据结构，确实不会有什么问题，如果是复杂的数据结构可能就会有问题了（**使用`AtomicReference`可以把`C A S`使用在对象上**），以链表数据结构为例，两个线程通过`C A S`去删除头节点，假设现在链表有`A->B`节点

![Image](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8ny1VNcSscicjAax5qNibFxqiabJLQZYt6OMXoBHbMIlLoNjgVt85LZlT0FGAoWB09ScvI5KITMSr9qxg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- 线程`1`删除`A`节点，`B`节点成为头节点，正要执行`C A S(A,A,B)`时，时间片用完，切换到线程`2`
- 线程`2`删除`A、B`节点
- 线程`2`加入`C、A`节点，链表节点变成`A->C`
- 线程`1`重新获取时间片，执行`C A S(A,A,B)`
- 丢失`C`节点

要解决`A B A`问题也非常简单，只要追加版本号即可，每次改变时加`1`，即`A —> B —> A`，变成`1A —> 2B —> 3A`，在`Java`中提供了`AtomicStampedRdference`可以实现这个方案

# 自旋锁

## CAS自旋时间太长

当一个线程获取锁时失败，不进行阻塞挂起，而是间隔一段时间再次尝试获取，直到成功为止，这种循环获取的机制被称为自旋锁(**`spinlock`**)。

自旋锁好处是，持有锁的线程在短时间内释放锁，那些等待竞争锁的线程就不需进入阻塞状态（**无需线程上下文切换/无需用户态与内核态切换**），它们只需要等一等（**自旋**），等到持有锁的线程释放锁之后即可获取，这样就避免了用户态和内核态的切换消耗。

自旋锁坏处显而易见，线程在长时间内持有锁，等待竞争锁的线程一直自旋，即CPU一直空转，资源浪费在毫无意义的地方，所以一般会限制自旋次数。

## 自旋锁基于CAS实现

最后来说自旋锁的实现，实现自旋锁可以基于`C A S`实现，先定义`lockValue`对象默认值`1`，`1`代表锁资源空闲，`0`代表锁资源被占用，代码如下

```java
public class SpinLock {   
    //lockValue 默认值1
    private AtomicInteger lockValue = new AtomicInteger(1);
    
    //自旋获取锁
    public void lock(){
        // 循环检测尝试获取锁
        while (!tryLock()){
            // 空转
        }
    }
    
    //获取锁
    public boolean tryLock(){
        // 期望值1，更新值0，更新成功返回true，更新失败返回false
        return lockValue.compareAndSet(1,0);
    }
    
    //释放锁
    public void unLock(){
        if(!lockValue.compareAndSet(0,1)){
            throw new RuntimeException("释放锁失败");
        }
    }

}
```

上面定义了`AtomicInteger`类型的`lockValue`变量，`AtomicInteger`是`Java`基于`C A S`实现的`Integer`原子操作类，还定义了3个函数`lock、tryLock、unLock`

tryLock函数-获取锁

- **期望值1，更新值0**
- **`C A S`更新**
- **如果期望值与`lockValue`值相等，则`lockValue`值更新为`0`，返回`true`，否则执行下面逻辑**
- **如果期望值与`lockValue`值不相等，不做任何更新，返回`false`**

unLock函数-释放锁

- **期望值`0`，更新值`1`**
- **`C A S`更新**
- **如果期望值与`lockValue`值相等，则`lockValue`值更新为`1`，返回`true`，否则执行下面逻辑**
- **如果期望值与`lockValue`值不相等，不做任何更新，返回`false`**

lock函数-自旋获取锁

- **执行`tryLock`函数，返回`true`停止，否则一直循环**

![Image](https://mmbiz.qpic.cn/mmbiz_png/23OQmC1ia8ny1VNcSscicjAax5qNibFxqiabQHKscYW1MI87ic7kFeJjDqKWIqRl72I4nPTwJ8OBaUBz7L0sN4FFBqA/640)

从上图可以看出，只有`tryLock`成功的线程（**把`lockValue`更新为`0`**），才会执行代码块，其他线程个`tryLock`自旋等待`lockValue`被更新成`1`，`tryLock`成功的线程执行`unLock`（**把`lockValue`更新为`1`**），自旋的线程才会`tryLock`成功。

