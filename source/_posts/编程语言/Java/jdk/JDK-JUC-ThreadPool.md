---
title: jdk线程池源码解析
date: 2020-10-17 19:52:16
categories: [java,oom]
tags:
---

### 类图

- ThreadPoolExecutor继承AbstractExecutorService, 提供了execute(Runnable)方法的实现

- AbstractExecutorService实现ExecutorService，提供了submit(Runnable)方法和submit(Callable)方法的实现
- ExecutorService继承了Executor接口，定义了submit(Runnable)方法和submit(Callable)方法
- Executor接口，定义execute(Runnable)方法

![img](https://filescdn.proginn.com/335ad046a09617ef5054d1336967556e/2ffc7547e74b396c2adc6cf790e4c316.webp)

### 创建线程池

无论哪种类型的线程池，最终都是直接或者间接通过ThreadPoolExecutor这个类来实现的。而ThreadPoolExecutor的有多个构造方法，最终都是调用含有7个参数的构造函数。

在这个构造方法中：

- 首先对corePoolSize（核心线程数）、maximumPoolSize（最大线程数）、keepAliveTime（核心线程以外的线程空闲时,存活的最大时间）、workQueue（线程队列）、threadFactory（线程工厂）、handler（拒绝策略）参数做了合法性校验；
- 给线程池的属性赋值；

### 提交任务

一个线程池可以接受任务类型有Runnable和Callable，我们以Runnable为例

#### ExecutorService.submit(Runnable)提交任务

AbstractExecutorService实现了submit方法

<img src="https://pic1.zhimg.com/80/v2-56cc90e0ec1c64ae28709ba2aba2dc34_720w.jpg" alt="img" style="zoom: 67%;" />

> RunnableFuture是Runnable的一个子类

**补充知识：线程状态的表示**

Doug Lea 采用一个 32 位的整数来存放线程池的状态和当前池中的线程数，其中高 3 位用于存放线程池状态，低 29 位表示线程数。我们知道，java 语言在整数编码上是统一的，都是采用补码的形式

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

// 这里 COUNT_BITS 设置为 29(32-3)，意味着前三位用于存放线程状态，后29位用于存放线程数
// 很多初学者很喜欢在自己的代码中写很多 29 这种数字，或者某个特殊的字符串，然后分布在各个地方，这是非常糟糕的
private static final int COUNT_BITS = Integer.SIZE - 3;

// 000 11111111111111111111111111111
// 这里得到的是 29 个 1，也就是说线程池的最大线程数是 2^29-1=536870911
// 以我们现在计算机的实际情况，这个数量还是够用的
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// 我们说了，线程池的状态存放在高 3 位中
// 运算结果为 111跟29个0：111 00000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;
// 000 00000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 001 00000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;
// 010 00000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;
// 011 00000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;

// 将整数 c 的低 29 位修改为 0，就得到了线程池的状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 将整数 c 的高 3 为修改为 0，就得到了线程池中的线程数
private static int workerCountOf(int c)  { return c & CAPACITY; }

private static int ctlOf(int rs, int wc) { return rs | wc; }

/*
 * Bit field accessors that don't require unpacking ctl.
 * These depend on the bit layout and on workerCount being never negative.
 */

private static boolean runStateLessThan(int c, int s) {
    return c < s;
}

private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}

private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
```

在这里，介绍下线程池中的各个状态和状态变化的转换过程：

- RUNNING：这个没什么好说的，这是最正常的状态：接受新的任务，处理等待队列中的任务
- SHUTDOWN：不接受新的任务提交，但是会继续处理等待队列中的任务
- STOP：不接受新的任务提交，不再处理等待队列中的任务，中断正在执行任务的线程
- TIDYING：所有的任务都销毁了，workCount 为 0。线程池的状态在转换为 TIDYING 状态时，会执行钩子方法 terminated()
- TERMINATED：terminated() 方法结束后，线程池的状态就会变成这个

> RUNNING 定义为 -1，SHUTDOWN 定义为 0，其他的都比 0 大，所以等于 0 的时候不能提交任务，大于 0 的话，连正在执行的任务也需要中断。

各个状态的转换过程有以下几种：

- RUNNING -> SHUTDOWN：当调用了 shutdown() 后，会发生这个状态转换，这也是最重要的
- (RUNNING or SHUTDOWN) -> STOP：当调用 shutdownNow() 后，会发生这个状态转换，这下要清楚 shutDown() 和 shutDownNow() 的区别了
- SHUTDOWN -> TIDYING：当任务队列和线程池都清空后，会由 SHUTDOWN 转换为 TIDYING
- STOP -> TIDYING：当任务队列清空后，发生这个转换
- TIDYING -> TERMINATED：这个前面说了，当 terminated() 方法结束后

#### Executor.execute(Runnnable )执行任务

ThreadPoolExecutor实现了execute(Runnable)方法

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    // 前面说的那个表示 “线程池状态” 和 “线程数” 的整数
    int c = ctl.get();

    // step1: 如果当前线程数少于核心线程数，那么直接添加一个 worker 来执行任务，
    if (workerCountOf(c) < corePoolSize) {
        // 添加任务成功，那么就结束了；返回 false 代表线程池不允许提交任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 到这里说明，要么当前线程数大于等于核心线程数，要么刚刚 addWorker 失败了

    // step2: 如果线程池处于 RUNNING 状态，把这个任务添加到任务队列 workQueue 中
    if (isRunning(c) && workQueue.offer(command)) {
        /* 这里面说的是，如果任务进入了 workQueue，我们是否需要开启新的线程
         * 因为线程数在 [0, corePoolSize) 是无条件开启新的线程
         * 如果线程数已经大于等于 corePoolSize，那么将任务添加到队列中，然后进到这里
         */
        int recheck = ctl.get();
        // 如果线程池已不处于 RUNNING 状态，那么移除已经入队的这个任务，并且执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 如果线程池还是 RUNNING 的，并且线程数为 0，那么开启新的线程
        // 到这里，我们知道了，这块代码的真正意图是：担心任务提交到队列中了，但是线程都关闭了
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // step3:如果 workQueue 队列满了，创建新的非核心worker
    else if (!addWorker(command, false))
        reject(command); // addWorker失败，说明当前线程数已经达到 maximumPoolSize，执行拒绝策略
}
```

step1：当一个任务被提交时，若线程池中的工作线程 < 核心线程，调用addWorker方法创建新的线程

step2：当前线程池中核心线程已满，但是任务队列workQueue未满，则添加;

step3：当前线程池中核心线程已满，任务队列也满了，就尝试调用addWorker创建新的线程（非核心），如果创建线程失败则执行拒绝策略；

##### step1：addWorker创建核心worker线程

addWorker()所创建的线程Worker，会在KeepLive销毁之前，一直调用循环方法getTask()，从workQueue队列获取任务。

```java
// 第一个参数是准备提交给这个线程执行的任务，之前说了，可以为 null
// 第二个参数为 true 代表使用核心线程数 corePoolSize 作为创建线程的界限，也就说创建这个线程的时候，
//         如果线程池中的线程总数已经达到 corePoolSize，那么不能响应这次创建线程的请求
//         如果是 false，代表使用最大线程数 maximumPoolSize 作为界限
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 这个非常不好理解
        // 如果线程池已关闭，并满足以下条件之一，那么不创建新的 worker：
        // 1. 线程池状态大于 SHUTDOWN，其实也就是 STOP, TIDYING, 或 TERMINATED
        // 2. firstTask != null
        // 3. workQueue.isEmpty()
        // 简单分析下：
        // 还是状态控制的问题，当线程池处于 SHUTDOWN 的时候，不允许提交任务，但是已有的任务继续执行
        // 当状态大于 SHUTDOWN 时，不允许提交任务，且中断正在执行的任务
        // 多说一句：如果线程池处于 SHUTDOWN，但是 firstTask 为 null，且 workQueue 非空，那么是允许创建 worker 的
        // 这是因为 SHUTDOWN 的语义：不允许提交新的任务，但是要把已经进入到 workQueue 的任务执行完，所以在满足条件的基础上，是允许创建新的 Worker 的
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 如果成功，那么就是所有创建线程前的条件校验都满足了，准备创建线程执行任务了
            // 这里失败的话，说明有其他线程也在尝试往线程池中创建线程
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 由于有并发，重新再读取一下 ctl
            c = ctl.get();
            // 正常如果是 CAS 失败的话，进到下一个里层的for循环就可以了
            // 可是如果是因为其他线程的操作，导致线程池的状态发生了变更，如有其他线程关闭了这个线程池
            // 那么需要回到外层的for循环
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    /*
     * 到这里，我们认为在当前这个时刻，可以开始创建线程来执行任务了，
     */

    // worker 是否已经启动
    boolean workerStarted = false;
    // 是否已将这个 worker 添加到 workers 这个 HashSet 中
    boolean workerAdded = false;
    Worker w = null;
    try {
        final ReentrantLock mainLock = this.mainLock;
        // 把 firstTask 传给 worker 的构造方法
        w = new Worker(firstTask);
        // 取 worker 中的线程对象，Worker的构造方法会调用 ThreadFactory 来创建一个新的线程
        final Thread t = w.thread;
        if (t != null) {
            // 整个线程池的全局锁，因为关闭一个线程池需要这个锁，至少持有锁的期间，线程池不会被关闭
            mainLock.lock();
            try {

                int c = ctl.get();
                int rs = runStateOf(c);

                // 小于 SHUTTDOWN 那就是 RUNNING，这个自不必说，是最正常的情况
                // 如果等于 SHUTDOWN，前面说了，不接受新的任务，但是会继续执行等待队列中的任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // worker 里面的 thread 可不能是已经启动的
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    // 加到 workers 这个 HashSet 中
                    workers.add(w);
                    int s = workers.size();
                    // largestPoolSize 用于记录 workers 中的个数的最大值
                    // 因为 workers 是不断增加减少的，通过这个值可以知道线程池的大小曾经达到的最大值
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 添加成功的话，启动这个线程
            if (workerAdded) {
                // 启动线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 如果线程没有启动，需要做一些清理工作，如前面 workCount 加了 1，将其减掉
        if (! workerStarted)
            addWorkerFailed(w);
    }
    // 返回线程是否启动成功
    return workerStarted;
}
```

三个关键点：

- 创建一个worker对象

  - ```
     w = new Worker(firstTask);
    ```

- 创建成功后，对线程池状态判断成功后，就去执行该worker对象的thread的启动。也就是说在这个方法里面启动了一个关联到worker的线程

  - ```
    final Thread t = w.thread;
    ```

- 线程启动

  - ```
    t.start();
    ```

  

###### ThreadPoolExecutor的内部类worker：真正干活的线程

ThreadPoolExecutor的内部类 Worker，因为 Doug Lea 把线程池中的线程包装成了一个个 Worker，放在HashSet<Worker> workers中。所以worker中，任务是 Runnable（内部变量名叫 task 或 command），线程是 Thread。

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
    private static final long serialVersionUID = 6138294804551838833L;

    final Thread thread; // 这个是真正的线程，任务靠你啦!!!

    // 前面说了，这里的 Runnable 是任务。为什么叫 firstTask？因为在创建线程的时候，如果同时指定了
    // 这个线程起来以后需要执行的第一个任务，那么第一个任务就是存放在这里的(线程可不止执行这一个任务)
    // 当然了，也可以为 null，这样线程起来了，自己到任务队列workQueue中取任务（getTask 方法）就行了
    Runnable firstTask;

    volatile long completedTasks; // 用于存放此线程完成的任务数，注意了，这里用了 volatile，保证可见性

    // Worker 只有这一个构造方法，传入 firstTask，也可以传 null
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this); // 调用 ThreadFactory 来创建一个新的线程
    }

    // 这里调用了外部类的 runWorker 方法
    public void run() {
        runWorker(this);
    }

    ...// 其他几个方法没什么好看的，就是用 AQS 操作，来获取这个线程的执行权，用了独占锁
}
```

Worker 这里又用到了抽象类 AbstractQueuedSynchronizer。<!--题外话，AQS 在并发中真的是到处出现，而且非常容易使用，写少量的代码就能实现自己需要的同步方式-->

从源码上看Worker这个类实现了Runnable接口

- 构造函数：线程工厂创建线程，把自己这个Worker实例传给线程 <!--因为worker就是一个Runnable-->
- [addWorder]()这个方法会调用Worker的执行线程t ==> t.start(), 那么就会启动[Worker中的run()方法]() <!--废话-->
- [Worker封装了任务Runnable和执行线程t]()！！！

###### ThreadPoolExecutor.runWorker(): 真正干活的方法

worker中的run()：委托给ThreadPoolExecutor.runWorker()方法

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run(); //task运行在当前线程中！！！
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

3个关键点：

- 里面有个循环，循环的条件是

```java
 while (task != null || (task = getTask()) != null) 
```

- 假如task不为空，进入循环会执行任务，并且可以看到，是在当前线程执行task


```text
task.run();
```

- 继续再往下finally代码块：代码的中task被重新赋值为空，然后执行下一循环；


```java
finally {
    task = null;
    w.completedTasks++;
    w.unlock();
}
```



###### runWorker()中的循环判断getTask(): 真正从任务队列消费任务和复用线程

getTask()方法才是工作线程Worker能够不断从任务队列消费任务的原因，保证Worker不断运行。

> 此方法有三种可能：
>
> 1. take方式取任务的特点是从缓存队列中取任务，若队列为空,则进入阻塞状态，阻塞直到获取到任务返回。默认 corePoolSize 之内的线程是不会被回收的，它们会一直等待任务
> 2. poll方式取任务的特点是从缓存队列中取任务,最长等待keepAliveTime的时长，取不到返回null。keepAliveTime 起作用的时候，也就是如果这么多时间内都没有任务，那么应该执行关闭
> 3. 如果发生了以下条件，此方法必须返回 null:
>       - 池中有大于 maximumPoolSize 个 workers 存在(通过调用 setMaximumPoolSize 进行设置)
>    - 线程池处于 SHUTDOWN，而且 workQueue 是空的，这种不再接受新的任务
>    - 线程池处于 STOP，不仅不接受新的线程，连 workQueue 中的线程也不再执行

```java

private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
       
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            // CAS 操作，减少工作线程数
            decrementWorkerCount();
            return null;
        }

        boolean timed;      // Are workers subject to culling?
        for (;;) {
            int wc = workerCountOf(c);
            // 允许核心线程数内的线程回收，或当前线程数超过了核心线程数，那么有可能发生超时关闭
            timed = allowCoreThreadTimeOut || wc > corePoolSize;

            // 这里 break，是为了不往下执行后一个 if (compareAndDecrementWorkerCount(c))
            // 两个 if 一起看：如果当前线程数 wc > maximumPoolSize，或者超时，都返回 null
            // 那这里的问题来了，wc > maximumPoolSize 的情况，返回 null 意味着关闭线程，因为有可能开发者调用了 setMaximumPoolSize() 将线程池的 maximumPoolSize 调小了，那么多余的 Worker 就需要被关闭
            if (wc <= maximumPoolSize && ! (timedOut && timed))
                break;
            if (compareAndDecrementWorkerCount(c))
                return null;
            c = ctl.get();  // Re-read ctl
            // compareAndDecrementWorkerCount(c) 失败，线程池中的线程数发生了改变
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
        // wc <= maximumPoolSize 同时没有超时
        try {
            // 到 workQueue 中获取任务
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            // 如果此 worker 发生了中断，采取的方案是重试
            // 解释下为什么会发生中断，这个读者要去看 setMaximumPoolSize 方法。

            // 如果开发者将 maximumPoolSize 调小了，导致其小于当前的 workers 数量，
            // 那么意味着超出的部分线程要被关闭。重新进入 for 循环，自然会有部分线程会返回 null
            timedOut = false;
        }
    }
}
```

这个方法中重点看下Runnable getTask()中的这段代码

```java
 Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
```

从队列中获取任务：[循环从队列中获取任务并赋值给task,然后调用task的run方法，以此循环]()。这样对于处理队列里的任务就不需要重新new 一个线程来处理了，因此实现了线程的重复利用！！！



##### step2：新的任务被提交到了队列中

 执行workQueue.offer(command)



##### step3: addWorker创建非核心worker线程





##### 总结：

|                   |                                                 |
| ----------------- | ----------------------------------------------- |
| execute()         | 1. coreSize未达到，addWorker(command)           |
|                   | 2. 超过coreSize, addWorker(null)                |
|                   | 3. 超过队列，addWorker(null)                    |
| addWorker()       | 1. 判断线程池状态                               |
|                   | 2. new Worker(commad); w.start()                |
| Worker.class      | 1. 构造方法: new Thread(Worker);                |
|                   | 2. runWorker(Worker);                           |
| runWorker(Worker) | 1.Runnable = Worker.firstTask                   |
|                   | 2.  While(task = getTask() != null); task.run() |
|                   |                                                 |



Worker本质上是个工作线程Thread + 任务Runnable。我一开始以为Worker是个干活的，就应该只是个Thread，没有必要封装任务Runnable。 再者，由于受到之前的固化思维，以为Thread和Runnable必须在同一行代码中，`new Thread(Runnable).start()`，所以Worker作为一个Thread，已经运行过`new Thread(Runnable).start()`，这个Thread和一个Runnbale已经绑定了，怎么还能获取到其他Runnable，并且运行呢？其实，这就是对 `Thread(Runnable).start()`和 `Runnable().run()`没有理解透彻。虽然Worker已经被运行了Thread(Runnable).start()，该worker的Thread已经和自己的Runnbale已经绑定了（这个Runnable其实就是firstTask，就是submit(Runnable)的时候那个Runnable），但是该worker的Runnable.run()方法，包含了getTask()方法从而获取其他Runnable, 然后执行`Runnable().run()`，这样Runnable就运行在当前线程了，当前线程是谁呢？当然是该worker的Thread了。

| 线程池pool所在的主线程 | worker所在的子线程                                           |
| ---------------------- | ------------------------------------------------------------ |
| submit(Runnable)       |                                                              |
| addWorker(Runnable)    |                                                              |
|                        | worker.start()调用worker.run()                               |
|                        | worker.run() 调用 ThreadPoolExecutor.runWorker(Worker this)  |
|                        | ThreadPoolExecutor.runWorkerr(Worker this)调用getTask()获取task |
|                        | task.run()在当前线程执行，即worker线程                       |

