---
title: JDK-lang-Thread
date: 2021-05-28 15:09:45
tags:
---



## 线程状态

### 操作系统中线程状态

传统的进（线）程状态一般划分如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfednyicQRNkh1ibicsEjORXmQPvuebO2CAvYA39TwmEx3t5agr9HlYDO9wxvRoW8eGut9tzZUGvvJyNg/640)

### jdk中线程状态

#### Runnable

##### 包含OS的ready和running状态

runnable 状态实质上是包括了 ready 状态和running 状态。

> A thread in the runnable state is executing in the Java virtual machine but it may be waiting for other resources from the operating system such as processor.

通常，Java的线程状态是服务于监控的，如果线程切换得是如此之快，那么区分 ready 与 running 就没什么太大意义了。

现今主流的 JVM 实现都把 Java 线程一一映射到操作系统底层的线程上，把调度委托给了操作系统，我们在虚拟机层面看到的状态实质是对底层状态的映射及包装。JVM 本身没有做什么实质的调度，把底层的 ready 及 running 状态映射上来也没多大意义，因此，统一成为runnable 状态是不错的选择。

##### 包含OS中由于IO引起的wait状态

一旦线程中执行到 I/O 有关的代码，相应线程立马被切走，然后调度 ready 队列中另一个线程来运行。这时执行了 I/O 的线程就不再运行，处于上图中的 waiting 状态。

而当 I/O 完成时，则用一种叫中断 （interrupt）的机制来通知 cpu：

cpu 会收到一个比如说来自硬盘的Interrput信号，并进入中断处理例程，CPU正在执行的线程因此被Interrput，回到 ready 队列。而先前因 I/O 而waiting 的线程随着 I/O 的完成也再次回到 ready 队列，这时 cpu 可能会选择它来执行。

> 进行阻塞式 I/O 操作时，Java 的线程状态究竟是什么？是 BLOCKED？还是 WAITING？

其实状态还是 RUNNABLE。

网络阻塞时同理，比如socket.accept，我们说这是一个“阻塞式(blocked)”式方法，但线程状态还是 RUNNABLE。

> 至少我们看到了，进行传统上的 IO 操作时，口语上我们也会说“阻塞”，但这个“阻塞”与线程的 BLOCKED 状态是两码事！

##### 总结

首先还是前面说的，注意分清两个层面：

- JVM层面：当进行阻塞式的 IO 操作时，或许底层的操作系统线程确实处在阻塞状态，但我们关心的是 JVM 的线程状态。JVM 把那些都视作资源，cpu 也好，硬盘，网卡也罢，有东西在为线程服务，它就认为线程在“执行”。

- 操作系统：操作系统的线程状态是围绕着 cpu 这一核心去述说的，这与 JVM 的侧重点是有所不同的。

前面我们也强调了“Java 线程状态的改变通常只与自身显式引入的机制有关”，如果 JVM 中的线程状态发生改变了，通常是自身机制引发的。

> 比如 synchronize 机制有可能让线程进入BLOCKED 状态，sleep，wait等方法则可能让其进入 WATING 之类的状态。

它与传统的线程状态的对应可以如下来看：

<img src="https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfednyicQRNkh1ibicsEjORXmQPEQSACKpCTzaaLPDxT4MNqw0KgSqLFX6LxM8qFbf61zybHibUGM8n3Cw/640" alt="图片" style="zoom: 50%;" />

RUNNABLE 状态对应了传统的 ready， running 以及部分的 waiting 状态。



#### BLOCKED: 阻塞状态

等待锁的释放，比如线程A进入了一个synchronized方法，线程B也想进入这个方法，但是这个方法的锁已经被线程A获取了，这个时候线程B就处于BLOCKED状态

#### WAITING: 等待状态

处于等待状态的线程是由于执行了3个方法中的任意方法。 

1. Object.wait() with no timeout
2. Thread.join() with no timeout
3. LockSupport.park()

处于waiting状态的线程会等待另外一个线程处理特殊的行为。 

再举个例子，如果一个线程调用了一个对象的wait方法，那么这个线程就会处于waiting状态直到另外一个线程调用这个对象的notify或者notifyAll方法后才会解除这个状态

> 比如线程处于WAITING状态，这个时候可以分析一下notifyAll或者signalAll方法的调用时机是否不对。

#### TIMED_WAITING: 

有等待时间的等待状态，比如调用了以下几个方法中的任意方法，并且指定了等待时间，线程就会处于这个状态。  

1. Thread.sleep(long millis)

2. Object.wait(long) with timeout 
3. Thread.join(long) with timeout
4. LockSupport.parkNanos(Object blocker, long deadline)
5. LockSupport.parkUntil(long deadline)



#### TERMINATED

线程中止的状态，这个线程已经完整地执行了它的任务



### java线程不同状态之间的转换



![img](http://f1.babyitellyou.com/sp1/img/20201206/1ed0ebb5d7a15469d852e640d8daa0dd.png)



#### RUNNABLE -> BLOCKED状态

代码执行到synchronized代码块和synchronized方法 时候要获取到锁，就变成BLOCKED状态。

> **我们平时所谓的 Java 在调用阻塞式 API 时，线程会阻塞，指的是操作系统线程的状态，并不是 Java 线程的状态。**

#### RUNNABLE -> WAITING状态

根据官方文档，线程执行到代码Object#wait()，Thread.join，LockSupport#park()时候线程会转移到该前状态。

#### RUNNABLE -> TIMED_WAITING状态

根据定义线程执行到Thread.sleep(long millis)，Object.wait(long timeout)，Thread.join(long millis)，LockSupport.parkNanos(Object blocker, long deadline)，LockSupport.parkUntil(long deadline)，和waitting相比主要是有个等待时间，且比waiting多了个sleep操作。

#### BLOCKED -> RUNNABLE状态

线程获得 监视器锁(synchronized 隐式锁)时，就又会从 BLOCKED 转换到 RUNNABLE 状态

#### WAITING -> RUNNABLE状态

对于Object.wait()，其它线程执行了同一个对象的Object#notify()或者nofityAll()时候唤醒线程，线程由WAITING -> RUNNABLE。

对于Thread.join()，其它线程执行完毕或者抛出异常后唤醒线程，线程由WAITING -> RUNNABLE。

对于LockSupport#park，在别处执行了此Thread的LockSuport#unPark时候唤醒线程，线程由WAITING -> RUNNABLE。

#### TIMED_WAITING -> RUNNABLE状态

对于Object.wait(long timeout)，线程阻塞timeout时间后，或者在timeout时间内其它线程执行了同一个对象的Object#notify()或者nofityAll()时候唤醒线程，线程由WAITING -> RUNNABLE。

对于Thread.join(long millis)，线程阻塞millis时间后，或者millis时间内其它线程执行完毕或者抛出异常后唤醒线程，线程由WAITING -> RUNNABLE。

对于LockSupport.parkNanos(Object blocker, long deadline)和LockSupport.parkUntil(long deadline)，线程阻塞deadline时间后，或者在别处执行了此Thread的LockSuport#unPark时候唤醒线程，线程由WAITING -> RUNNABLE。

Thread.sleep(long millis)，休眠millis时间后线程自动唤醒，或者线程被打断休眠，也会被唤醒。线程由WAITING -> RUNNABLE。



## jstack显示的java线程状态

```
jstack PID(10进制)
```

生成线程堆栈文件

> java.lang.Thread.State: BLOCKED 线程处于阻塞状态，遇到了synchronized处于阻塞状态
>
> java.lang.Thread.State: RUNNABLE 线程处于正运行状态，或者阻塞在IO状态 
>
> java.lang.Thread.State: TIMED_WAITING (on object monitor) Object.wait(long)加超时时间操作 java.lang.Thread.State: TIMED_WAITING (parking) 通过LockSupport.parkXXX加超时时间操作 java.lang.Thread.State: TIMED_WAITING (sleeping) 通过Thread.sleep加休眠时间操作 
>
> java.lang.Thread.State: WAITING (on object monitor) Object.wait()无超时时间操作 
>
> java.lang.Thread.State: WAITING (parking) 通过LockSupport.park())操作



### 线程中断

除了Object.nofity()，nofityAll()，LockSupport.unPark(Thread)操作唤醒线程或者超时自动唤醒线程，工作中也经常使用到interrupt() 方法对阻塞中的线程(WAITING、TIMED_WAITING 状态)进行中断，以便使线程从阻塞(WAITING、TIMED_WAITING 状态)中被唤醒。interrupt()操作是给正在处于阻塞状态的线程发个中断通知，以便线程从阻塞中唤醒过来。

线程 A 处于 WAITING、TIMED_WAITING 状态时，如果其他线程调用线程 A 的 interrupt() 方法，会使线程 A 返回到 RUNNABLE 状态(中间可能存在BLOCKED状态)，同时线程 A 的代码会触发 InterruptedException 异常。通过查看Object.wait()，Thread.join()方法上都有throws InterruptedException。这个异常的触发条件就是：其他线程调用了该线程的 interrupt() 方法。

当RUNNABLE状态线程在阻塞到IO操作时候，此时线程状态还是RUNNABLE ，但是实际在linux线程模型中是阻塞状态，比如线程A阻塞在 java.nio.channels.Selector.select() 上时，如果其他线程调用线程 A 的 interrupt() 方法，线程 A 的 java.nio.channels.Selector 会立即返回。

## 线程池内线程的状态

- 预热阶段，线程状态也是从NEW->RUNNABLE-BLOCKED/WAITING/TIMED_WAITING状态

- 线程池活动达到core线程后，接入的请求都会存放到队列内，如果队列内任务为空，那么Worker线程getTask()阻塞，处于waiting状态

![img](http://f1.babyitellyou.com/sp1/img/20201206/87d263be12c962900f8a33fa6b68e264.png)

其中代码@1和@2底层都是使用的LockSupport.park()挂起线程 <!--ReentrantLock锁--> ，因此，线程池内的线程空闲时候，线程状态是waiting状态。

验证如下:

```java
 public class AllocateServiceImpl implements AllocateService, InitializingBean{
 
     private ThreadPoolExecutor pool;
     
     @Override
     public void afterPropertiesSet() throws Exception {
         ThreadFactory threadFactory = new ThreadFactoryBuilder().
                 setNameFormat("my-pool-%d").build();
         pool = new ThreadPoolExecutor(10, 10, 60L, TimeUnit.MINUTES, 
                 new LinkedBlockingDeque<Runnable>(1024), threadFactory, new ThreadPoolExecutor.AbortPolicy());
         pool.prestartAllCoreThreads();//启动所有线程
     }
 }
```

启动后使用

```
jstack pid
```

验证截图如下

![img](http://f1.babyitellyou.com/sp1/img/20201206/8484f37a0899c7617f7fc4e9db310e82.png)

 

## JVM线程状态与cpu关系

> NEW 这线程刚创建，还未执行, CPU=0%
>
> RUNNABLE 线程处于可运行状态，但是实际可能是正在运行或者等待io资源，因此不能完全确定是否在使用cpu资源。 
>
> BLOCKED 线程阻塞状态，肯定不会使用cpu资源 
>
> WAITING 线程休眠状态，肯定不会使用cpu资源 
>
> TIMED_WAITING 线程休眠状态，肯定不会使用cpu资源

通过上面分析，在分析cpu100%问题时候，`jstack PID` 后，只需要查看dump结果中处于RUNNABLE状态的线程即可，但是处于RUNNABLE状态的线程，不代表就一定正在使用cpu资源，因此要特定分析（通过代码分析）。通常是通过top -Hp PID确定使用cpu资源高的线程后，再通过多次jstack操作，查看哪些线程堆栈一直处于RUNNABLE。

举例如下：

1.处于waiting，timed_waiting，blocked状态肯定不消化cpu，如下图，分析cpu时候忽略这些线程

![img](http://f1.babyitellyou.com/sp1/img/20201206/7cbd9e0e3d0429f3457a83cc93986736.png)

2.比如下面这个，虽然处于RUNNABLE状态，实际上是阻塞在监听接入连接上（阻塞在网络IO上），因此实际是不使用cpu资源

![img](http://f1.babyitellyou.com/sp1/img/20201206/aac05a22a9f161e09ff13ccd498d4e3c.png)

3.如下图这个，线程状态是RUNNABLE，这里是使用正则表达式解析，正在使用cpu资源

![img](http://f1.babyitellyou.com/sp1/img/20201206/ce38281f97a1a12abb4811f2b11f94d6.png)

 

## JVM线程状态与锁的关系

A. sleep、wait、join都会使线程进入阻塞状态(waiting/timed_waiting状态)，同时也都会释放cpu资源

> yield是释放cpu资源，然后又抢夺cpu资源，目的是为了让其它线程有机会获取cpu资源进行处理，但是线程状态还是runnable。

B. sleep如果是在锁方法内执行，比如同步代码块或者重入锁方法内执行，是不会释放锁。<!--真不要脸，自己不用CPU，还阻塞其他使用该锁的线程使用CPU-->而wait会释放锁。

- wait用于锁机制，sleep不是，这也是为什么sleep不释放锁，wait释放锁的原因，
- sleep是线程的方法，跟锁没关系，
- wait，notify，notifyall 都是Object对象的方法，是一起使用的，用于锁机制。

有个特殊的Thread.sleep(0)，操作这个动作是让出cpu，让其它线程又机会获取cpu资源执行

 

## 线程中断

### 3个中断相关的方法

 停止一个线程意味着在任务处理完任务之前停掉正在做的操作，也就是放弃当前的操作。

在java中有以下3种方法可以终止正在运行的线程：

1. 使用退出标志，使线程正常退出，也就是当run方法完成后线程终止。
2. 使用stop方法强行终止，但是不推荐这个方法，因为stop和suspend及resume一样都是过期作废的方法。
3. 使用interrupt方法中断线程。

#### Method1: interrupt()中断线程(软中断)

> Interrupts this thread.
> Unless the current thread is interrupting itself, which is always permitted, the checkAccess method of this thread is invoked, which may cause a SecurityException to be thrown.
>
> If this thread is blocked in an invocation of the wait(), wait(long), or wait(long, int) methods of the Object class, or of the join(), join(long), join(long, int), sleep(long), or sleep(long, int), methods of this class, then its interrupt status will be cleared and it will receive an InterruptedException.

调用interrupt方法是在当前线程中打了一个停止标志，并不是真的停止线程，无法硬中断线程

```java
public class MyThread extends Thread {
    public void run(){
        super.run();
        for(int i=0; i<500000; i++){
            System.out.println("i="+(i+1));
        }
    }
}

public class SpringBootApplication {
    public static void main(String args[]){
        Thread thread = new MyThread();
        thread.start();
        try {
            Thread.sleep(2000);
            thread.interrupt();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：可以看到线程继续运行，并没有中断

```
...
i=499998
i=499999
i=500000
```

#### Method2: 静态方法interrupted()判断当前线程是否中断状态

> Tests whether the current thread has been interrupted. <font color='red'>The interrupted status of the thread is cleared by this method. In other words, if this method were to be called twice in succession, the second call would return false</font>> (unless the current thread were interrupted again, after the first call had cleared its interrupted status and before the second call had examined it).
> A thread interruption ignored because a thread was not alive at the time of the interrupt will be reflected by this method returning false.
> Returns:
> true if the current thread has been interrupted; false otherwise.

```java
public static boolean interrupted{
	return currentThread().isInterrputed(true);
}
```

interrupted(): 测试<font color='red'>当前线程</font>是否已经中断，线程的中断状态由该方法清除。 换句话说，如果连续两次调用该方法，则第二次调用返回false。

```java
public class Run {
    public static void main(String args[]){
        Thread thread = new MyThread();
        thread.start();
        try {
            Thread.sleep(2000);
            thread.interrupt();

            System.out.println("stop 1??" + thread.interrupted());
            System.out.println("stop 2??" + thread.interrupted());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

运行结果：

```
stop 1??false
stop 2??false
```

类Run.java中虽然是在thread对象上调用以下代码：thread.interrupt(), 后面又使用

```
System.out.println("stop 1??" + thread.interrupted());
System.out.println("stop 2??" + thread.interrupted());
```

来判断thread对象所代表的线程是否停止，但从控制台打印的结果来看，线程并未停止，这也证明了interrupted()方法的解释，测试[当前线程]()是否已经中断。这个当前线程是main主线程，它从未中断过，而是子线程thread中断了，所以打印的结果是两个false.

如何使main线程产生中断效果呢？就是把thread.interrupt改为main线程interrupt <!--Thread.currentThread()获取当前main线程-->

```java
public class Run2 {
    public static void main(String args[]){
        Thread.currentThread().interrupt();
        System.out.println("stop 1??" + Thread.interrupted());
        System.out.println("stop 2??" + Thread.interrupted());
        System.out.println("End");
    }
}
```

运行效果为：

```
stop 1??true
stop 2??false
End
```

方法interrupted()的确判断出当前线程是否是停止状态。

#### Method3: isInterrupted()判断线程是否中断状态

> Tests whether this thread has been interrupted. <font color='red'>The interrupted status of the thread is unaffected by this method.</font>
> A thread interruption ignored because a thread was not alive at the time of the interrupt will be reflected by this method returning false.
> Returns:
> true if this thread has been interrupted; false otherwise.

```java
public boolean isInterrupted{
	return isInterrupted(false);
}

private native boolean isInterrupted(boolean ClearInterrupted)
```

isInterrupted()并<font color='red'>不清除中断状态</font>,所有连续2次调用并不清除中断状态

```java
public class Run3 {
    public static void main(String args[]){
        Thread thread = new MyThread();
        thread.start();
        thread.interrupt();
        System.out.println("stop 1??" + thread.isInterrupted());
        System.out.println("stop 2??" + thread.isInterrupted());
    }
}
```

运行结果：

```
stop 1??true
stop 2??true
```

### 捕获中断状态

#### 通过判断中断状态，抛出异常，停止线程

在线程中用for语句来判断一下线程是否是停止状态，如果是停止状态，则后面的代码不再运行即可

```java
public class MyThread extends Thread {
    public void run(){
        super.run();
        try {
            for(int i=0; i<500000; i++){
                if(this.interrupted()) {
                    System.out.println("线程已经终止， for循环不再执行");
                        throw new InterruptedException();
                }
                System.out.println("i="+(i+1));
            }

            System.out.println("这是for循环外面的语句，也会被执行");
        } catch (InterruptedException e) {
            System.out.println("进入MyThread.java类中的catch了。。。");
            e.printStackTrace();
        }
    }
}
```



> 建议使用“抛异常”的方法来实现线程的停止，因为在catch块中还可以将异常向上抛，使线程停止事件得以传播。
>





#### 通过捕获InterruptedException停止

如果线程在sleep()状态下停止线程，会是什么效果呢？

```java
public class MyThread extends Thread {
    public void run(){
        super.run();
        try {
            System.out.println("线程开始。。。");
            Thread.sleep(200000);
            System.out.println("线程结束。");
        } catch (InterruptedException e) {
            System.out.println("在沉睡中被停止, 进入catch， 调用isInterrupted()方法的结果是：" + this.isInterrupted());
            e.printStackTrace();
        }

    }
}
```

使用Run.java运行的结果是：

```
线程开始。。。
在沉睡中被停止, 进入catch， 调用isInterrupted()方法的结果是：false
java.lang.InterruptedException: sleep interrupted
 at java.lang.Thread.sleep(Native Method)
 at thread.MyThread.run(MyThread.java:12)
```

从打印的结果来看， 如果在sleep状态下停止某一线程，会进入catch语句，并且清除停止状态值，使之变为false。

前一个实验是先sleep然后再用interrupt()停止，与之相反的操作在学习过程中也要注意：

```java
public class MyThread extends Thread {
    public void run(){
        super.run();
        try {
            System.out.println("线程开始。。。");
            for(int i=0; i<10000; i++){
                System.out.println("i=" + i);
            }
            Thread.sleep(200000);
            System.out.println("线程结束。");
        } catch (InterruptedException e) {
             System.out.println("先停止，再遇到sleep，进入catch异常");
            e.printStackTrace();
        }

    }
}

public class Demo {
    public static void main(String args[]){
        Thread thread = new MyThread();
        thread.start();
        thread.interrupt();
    }
}
```

运行结果：

```
i=9998
i=9999
先停止，再遇到sleep，进入catch异常
java.lang.InterruptedException: sleep interrupted
 at java.lang.Thread.sleep(Native Method)
 at thread.MyThread.run(MyThread.java:15)
```

