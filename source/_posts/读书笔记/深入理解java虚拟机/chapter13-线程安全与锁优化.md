---
title: 'chapter13:线程安全与锁优化'
date: 2020-11-16 22:17:11
tags:[java]
---

ref:

1. Java并发-atomic原子类包源码剖析
   https://www.jianshu.com/p/27e07df2672e





1. 线程安全的定义：参考JCP中作者的定义

2. 线程安全的程度（很有实际借鉴的分类）

   1. 不可变对象，如Integer
   2. 绝对线程安全，“不论运行环境，调用者不需要**额外的同步措施**”。注意！！！这是不对的。比如java.util.Vecotr的类中,add()和get()等方法都被synchronized修饰，但是遇到线程A读操作，线程B删除操作，就会遇到Out of index的异常。所以线程A和线程B也需要被调用者加上额外的同步。
   3. 相对线程安全。同上，需要调用者加上额外的同步措施
   4. 线程兼容。类本身就不安全，所以，需要调用者加上额外的同步措施
   5. 线程对立。非常少见，Java已经在避免了。

3. 实现线程安全的方法

   1. 互斥同步：

      1. 定义：同步是指如果多个线程在同时访问共享数据时，只有一个线程可以访问。
      2. 同步是实现线程安全的目的；互斥是实现同步的手段；临界区、互斥量、信号量是主要的互斥实现方式。
      3. synchronized关键字：
         1. 编译之后，在同步代码块前后行程monitorenter和monitorexit两个字节码。这两个字节码都需要一个reference类型的参数来指明锁定和解锁的对象，若不指明，则使用对象实例或者Class对象
         2. 当前线程尝试获取**对象的锁**，获取锁，则锁的计数器+1；释放锁，则锁的计数器-1；
            获取锁失败，则阻塞等待。
         3. 注意：
            1. 可重入性
            2. JAVA线程是要映射到系统线程的，所以唤醒或者阻塞一个线程都需要操作系统，从用户态到内核态，耗费CPU。所以有经验的，都会在通知操作系统阻塞线程之前加入一段自旋等待过程，避免频繁切入到内核态
      4. ReentrantLock:
         1. 与synchronized相同 ==> 可重入
         2. 与synchronized不相同 ==> 代码写法上
            1. API层面的互斥锁，lock()与unlock()方法配合try/finally语法块来完成
            2. 原生语法层面上的互斥锁
               1. 等待可中断
               2. 公平锁
               3. 锁绑定多个条件

   2. 非阻塞同步

      1. 由于互斥同步是进行线程阻塞和唤醒，所以也叫阻塞同步，比较耗费性能，从处理问题的方式上来讲，也叫“悲观锁”，无论共享数据是否竞争，都必须加锁，否则都悲观地认为会出问题。所以现在使用“乐观锁”，也叫非阻塞同步

      2. 乐观并发，是先进行操作，再检查冲突；有冲突，再补救；无冲突，就成功；

         ==>所以，如果这个操作和检查两个步骤，依然需要软件的同步操作，那就是没必要了，所以需要依靠“硬件指令”

      3. 常见的硬件指令为CAS，这条指令需要3个操作数，分别是内存位置（V），旧的预期值（A）和新值（B）。CAS指令执行时，仅当V符合A是，处理器才用B更新V的值，否则不更新，无论是否更新，都会返回V。上述过程是个原子操作。

      4. JDK1.5以后，JAVA才使用CAS操作，由sun.misc.Unsafe类里面的方法包装，由虚拟机特殊处理，编译处理后，只有一条处理器CAS指令，没有方法调用的过程，所以可以认为是无条件内联进去了

      5. 场景

         1. JUC包中的AutoInteger类，其中的方法使用了Unsafe类的CAS操作。

            ```java
            public class Test { 
                public static void main(String args[]) {
                    int threadCnt = 20;
                    Thread[] threads = new Thread[threadCnt];
                    CountDownLatch cdt = new CountDownLatch(threadCnt);
                    for (int i = 0; i < threadCnt; i++) {
                        threads[i] = new Thread(new Runnable() {
                            @Override
                            public void run() {
                                for (int j = 0; j < 1000000; j++) {
                                    increase();
                                }
                                cdt.countDown();
                            }
                        });
                        threads[i].start();
                    }
                    for (Thread i : threads) {
                        try {
                            i.join();//保证前面的所有的线程执行玩再接着执行主线程
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
            
                    System.out.println(count + "   " + count2.get());
                }
            
                public static int count = 0;
                public static AtomicInteger count2 = new AtomicInteger(0);
            
                public static void increase() {
                    count++;
                    count2.incrementAndGet();
                }
            ```

            count一直小于2000W，而count2一直是2000W。如果把count设置为volatile,可以吗？因为volatile可以保证线程间的可见性以及禁止指令的重排序。答案是：不可以。因为count++, 不是原子操作，而且volatile不能保证原子性。而count2是原子类，可以保证count2++是原子操作。

            ```java
                /**
                 * Atomically increment by one the current value.
                 * @return the updated value
                 */
                public final int incrementAndGet() {
                    for (;;) {
                        int current = get();
                        int next = current + 1;
                        if (compareAndSet(current, next))
                            return next;
                    }
                }
            ```

            incrementAndGet（）方法在一个**无限循环**中（就是一个自旋锁），不断尝试将一个比当前值大1的新值赋给自己。如果失败了，那说明在执行“获取-设置”操作的时候值已经有了修改，于是再次循环进行下一次操作，直到设置成功为止。

   3. 无同步方案

      1. 可重入代码：纯代码，就是不涉及任何共享数据，比如堆上的数据或者公用的系统资源。判断是否是可重入代码的标准：输入相同参数，返回相同结果，就是可重入代码，也就是线程安全。
      2. 

