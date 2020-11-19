---
title: chapter2线程安全性
date: 2020-10-23 08:47:23
tags: [并发]
---



1. Race condition: 由于多线程之间不恰当的执行时序，而出现的不正确结果

   1. 比如: 没有同步的情况下，使用实例遍历count统计servlet的调用次数，count++就会出现race condition;
      这是一个"读取-修改-写入"的操作序列，并且结果依赖于之前的状态

   2. 最常见的race condition: Check Then Act（先检查后执行）

      1. 比如观察某个条件为真（比如文件X不存在），然后根据这个条件作出相应的动作（创建文件X）
      2. 但实际上观察 与 动作之间，观察的结果可能已经无效（另一个线程正在创建文件X）

   3. Check Then Act的一个示例：延迟初始化

      1. ```java
         //这段代码的执行代价很高
         public class LazyInitRace{
           private ExpensiveObject instance = null;
           public ExpensiveObject getInstance(){
             if(instance == null)
               instance = new ExpensiveObject();
             return instance;
           }
         }
         ```

      2. 线程A和线程B同时执行getInstance();

         1. A看到instance为空，则开始初始化
         2. B检查instance是否为空，取决于A是否初始化完成；如果A需要花很长时间初始化，那么B检查到instance是空，也开始初始化

2. 复合操作必须是原子操作

   1. 原子操作是指对于访问同一个状态的所有操作（包括操作本身），这个操作是一个以原子方式执行的操作。
      比如getInstance()方法中，所有代码都访问了instance这个状态，所以必须使用原子操作。
      这样带来的好处是，假定有两个操作A和B，如果从执行A的线程来看，当另一个线程执行B时，要么将B全部执行完，要么完全不执行B，那么A和B对彼此来说是原子的，也就没有race condition的发生。
      1. 要避免race condition, 就必须在某个线程修改该变量时，通过某种方式防止其他线程使用这个变量，从而确保其他线程要么在修改操作发生之前或者修改操作完成以后读取和修改状态，而不是在修改操作的进行当中。
         比如“先检查后执行”和“读取-修改-写入”等复合操作（包含了一组必须以原子方式执行的操作以确保线程安全性）必须是原子的。
      2. 比如，程序计数器，使用AtomicLong来代替long，能够确保所有对计数器状态的访问操作都是原子的。由于Servlet的状态就是计数器的状态，而且计数器的状态是线程安全的 ==> Servlet类也是线程安全的。

3. 类的线程安全

   1. 当无状态的类中添加一个状态时，如果该状态完全由线程安全的对象来管理，那么这个类就是线程安全的。
   2. 当状态变量由一个变为多个时，问题变得复杂
      1. 若希望向类中添加更多的状态变量，那么仅仅添加更多的线程安全的状态变量，是不可行的；因为尽管原子引用或者原子变量时线程安全，但是只要存在race condition, 就是线程不安全
      2. 线程安全性的定义：无论多个线程之间操作采用何种执行时序或者交替方式，都要保证不变性条件不被破坏。比如，lastFactor中缓存的因数之积 应该等于 lastNumber中缓存的数值，这个不变性条件必须保证。
      3. 当不变性条件涉及到多个状态变量时，那么状态变量之间产生约束了。所以，为了保证多个变量状态的一致性，必须在单个原子操作中，更新所有相关的状态变量。

4. java内置锁支持原子性

   1. 内置锁的定义
      
      1. 同步方法（以Class对象作为锁，被称为Intrinsic Lock or Monitor Lock）
      
   2. 程序在进入同步代码块时，自动获得锁；退出时，无论是正常还是异常退出，都自动释放；所以获得内置锁的唯一途径，就是进入同步代码块或者同步方法
   
   3. 内置锁相当于互斥锁，意味着最多只有一个线程持有该锁，所以由内置锁保护的代码块，是以原子方式进行。
      拓展：并发环境中的原子性 = 事务应用中的原子性：即，一组语句作为一个不可分割的单元被执行，任何一个执行同步代码块的线程，都不可能看到其他线程正在执行由同一个锁保护的同步代码块
      
   4. 比如，使用synchronized修饰servlet的service方法，以保证 lastFactor中缓存的因数之积=lastNumber中缓存的数值，这个不变性条件；但是！！！性能太差了，影响服务器响应！！！
   
   5. 内置锁的性质：重入
   
      1. 定义：当一个线程试图获取一个由他自己持有的锁时，那么这个请求是成功的
   
      2. 实现：为每个锁关联一个获取计数值和一个所有者线程。当线程请求一个未被持有的锁时，JVM将记下锁的持有者，并且将获取计数值=1，如果线程继续获取这个锁，则获取计数值++，当线程退出同步代码块时，则计数值--。为0时，意味着锁被释放
   
      3. 比如，子方法调用父方法
   
         ```java
         public class base{
         		public synchronized void do(){...}
         }
         
         public class child extends base{
         		public synchronized void do(){
         				System.out.println("...");
               	super.do();
         		}
         }
         ```

5. 锁如何保护共享状态

   1. 前提：什么样的数据才需要被锁保护？
      	只有被**多个线程 ** ***同时访问*** 的**可变数据**才需要通过锁来保护
   2. 原理:
          因为锁能使它保护的代码路径以串行的方式执行，则通过锁来实现一些协议以保证对共享状态的独享
   3. 上文提到了对共享状态访问的复合操作，如何使用锁保护由复合操作包裹的共享状态呢？
      1. 对复合操作加锁就成为同步代码块 => 可以使其成为原子操作，但是却 != 同步
      2. 对于可能被多个线程同时访问的可变状态变量，在访问它时都需要使用同一个锁，则称为“该状态变量由该锁保护的”，可见是锁保护了变量，不是复合操作或者同步代码块保护了变量，注意区别
   4. 每个共享的并且可变的变量，都必须只由一个锁来保护
   5. 当类的***不变性条件***涉及一个或者多个状态变量时
      1. **一个变量**时，由于访问必须获取锁，这样就确保在同一时刻只有一个线程可以访问这个变量
      2. **多个变量**时，需要添加额外的条件，这些状态变量必须交给同一个锁来保护
   6. 既然**同步**（我猜测作者指的是同步代码块和同步方法）**可以避免竞争条件**，为什么不在每个方法都声明synchronized?
      1. 如果滥用synchronized, 则会出现**过多的同步**
      2. 如果只是将每个方法作为同步方法，例如Vector,那么**并不足以确保Vector上复合操作都是原子的**，比如put-if-absent
   7. 常见的加锁约定：
      1. 所有的可变状态封装在对象内部
      2. 并通过对象的内置锁，对类内部所有访问可变状态的代码路径进行同步
      3. 通过以上2个步骤，使得在该对象上不会发生并发访问
      4. 注意：如果新的代码或者方法，没有使用同步，那么这种约定被破坏

6. synchronized方法的粗粒度特性，所带来的性能问题
   
   1. 以servlet的缓存为例, 由于service是一个synchronized方法，那么意味着每个时间内只有一个线程可以执行该方法，这就违背了设计servlet框架的初衷（servlet能同时处理多个请求）。同时，即使有多个空余的CPU，并且负载很高，但是由于每个时间内只有一个请求被处理，其他请求只能等待，意味着这些CPU依旧空闲。相当于把多线程变成了单线程。
      <img src="https://img-blog.csdn.net/20180913104158845?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0N1cnJ5Nzg5NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" style="zoom: 67%;" />

```java
   public class SynchronizedFactorizer extends GenericServlet implements Servlet {
       // 这里的 @GuardedBy 指的是被内置锁 synchronized 对象保护 没有实际意义，是一个语义化的注解
       @GuardedBy("this")
       private BigInteger lastNumber;
       @GuardedBy("this")
       private BigInteger[] lastFactors;
   		
     	// 使用内置锁修饰的方法，每次只有一个线程能执行，其他线程需要等待执行的线程方法调用完后释放锁
       @Override
       public synchronized void service(ServletRequest req, ServletResponse resp) {
           BigInteger i = extractFromRequest(req);
           if (i.equals(lastNumber)) {
               encodeIntoResponse(resp, lastFactors);
           } else {
               BigInteger[] factors = factor(i);
               lastNumber = i;
               lastFactors = factors;
               encodeIntoResponse(resp, factors);
           }
       }
   
       void encodeIntoResponse(ServletResponse resp, BigInteger[] factors) {
       }
   
       BigInteger extractFromRequest(ServletRequest req) {
           return new BigInteger("7");
       }
   
       BigInteger[] factor(BigInteger i) {
           // Doesn't really factor
           return new BigInteger[]{i};
       }
   }
```

   2. 如何解决上述问题：

        1. 降低锁的粒度，把需要由锁保护的代码或者说是变量与不需要锁保护的代码或者变量分开
             1. 比如局部变量（位于栈上），不会在多个线程间共享，所以不需要同步
   3. 新的代码
```java
public class CachedFactorizer extends GenericServlet implements Servlet {
    @GuardedBy("this") private BigInteger lastNumber;
    @GuardedBy("this") private BigInteger[] lastFactors;
    @GuardedBy("this") private long hits;
    @GuardedBy("this") private long cacheHits;

    public synchronized long getHits() {
        return hits;
    }

    public synchronized double getCacheHitRatio() {
        return (double) cacheHits / (double) hits;
    }

    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = null;
        synchronized (this) {
            ++hits;
            if (i.equals(lastNumber)) {
                ++cacheHits;
                factors = lastFactors.clone();
            }
        }
        if (factors == null) {
            factors = factor(i);
            synchronized (this) {
                lastNumber = i;
                lastFactors = factors.clone();
            }
        }
        encodeIntoResponse(resp, factors);
    }

    void encodeIntoResponse(ServletResponse resp, BigInteger[] factors) {
    }
}
```

注意：cacheHits不再使用AtomicLong类型(虽然也可以使用)，因为已经使用同步代码块来构成原子操作了，没有必要使用原子变量了，两种不同的同步机制会导致混乱。





Summary:

编写线程安全的并发程序，核心在于：在访问共享的可变状态时需要正确的管理。本章介绍了，通过同步避免多个线程在同一时间访问相同的数据

