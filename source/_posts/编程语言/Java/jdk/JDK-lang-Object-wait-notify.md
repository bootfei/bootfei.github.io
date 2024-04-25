---
title: JDK-lang-Thread
date: 2021-05-28 15:09:45
tags:
---





## JVM线程状态与锁的关系

sleep、wait、join都会使线程进入阻塞状态(waiting/timed_waiting状态)，同时也都会释放cpu资源

> yield是释放cpu资源，然后又抢夺cpu资源，目的是为了让其它线程有机会获取cpu资源进行处理，但是线程状态还是runnable。

sleep如果是在锁方法内执行，比如同步代码块或者重入锁方法内执行，是不会释放锁。而wait会释放锁。

- wait用于锁机制，sleep不是，这也是为什么sleep不释放锁，wait释放锁的原因，
- sleep是线程的方法，跟锁没关系，
- wait，notify，notifyall 都是Object对象的方法，是一起使用的，用于锁机制。

有个特殊的Thread.sleep(0)，操作这个动作是让出cpu，让其它线程又机会获取cpu资源执行

## wait()官方说明

The recommended approach to waiting is to check the condition being awaited in a while loop around the call to wait, as shown in the example below. Among other things, this approach avoids problems that can be caused by spurious wakeups.
 synchronized (obj) {      while (<condition does not hold> and <timeout not exceeded>) {          long timeoutMillis = ... ; // recompute timeout values          int nanos = ... ;          obj.wait(timeoutMillis, nanos);      }      ... // Perform action appropriate to condition or timeout  }

## notify()官方说明

Wakes up all threads that are waiting on this object's monitor. A thread waits on an object's monitor by calling one of the wait methods.
The awakened threads will not be able to proceed until the current thread relinquishes the lock on this object. The awakened threads will compete in the usual manner with any other threads that might be actively competing to synchronize on this object; for example, the awakened threads enjoy no reliable privilege or disadvantage in being the next thread to lock this object.
This method should only be called by a thread that is the owner of this object's monitor. See the notify method for a description of the ways in which a thread can become the owner of a monitor.
Throws:
IllegalMonitorStateException – if the current thread is not the owner of this object's monitor.



!["wait notify and notifyall in java synchronized](https://2.bp.blogspot.com/-7yTGxlOBfu4/VvUdKBO3RhI/AAAAAAAAFYE/tSNS9y3Gb2sGPE4nrnTvllraf0lYOt-SA/w615-h461/Wait%2Band%2BNotify%2BJava%2BConcurrency.jpg)



> Since the wait() method in Java also releases the lock prior to waiting and reacquires the lock prior to returning from the [wait() method](http://javarevisited.blogspot.com/2011/12/difference-between-wait-sleep-yield.html), we must use this lock to ensure that checking the condition (buffer is full or not) and setting the condition (taking element from the buffer) is atomic which can be achieved by using synchronized method or block in Java.
>
> Read more: https://javarevisited.blogspot.com/2011/05/wait-notify-and-notifyall-in-java.html#ixzz7w2kARh54

### 说明

无论是wait()还是notify()，其执行的前提条件都是：该线程首先持有资源对象object的锁。否则的话，就会抛出IllegalMonitorStateException异常。
 `wait(long timeout)`：和wait()方法类似的还有一个wait(long timeout)，如果线程一直没有被唤醒，当超时时间到了，就会自动被唤醒。
 `notifyAll()`：notify()方法，也对应有一个notifyAll()方法，该方法会将等待队列里的所有线程都唤醒，唤醒后的线程行为和notify()保持一致。

事实上一个线程被唤醒的方式包括以下几种：

- 其他线程调用object的notify()方法， 而该线程刚好被从等待队列中选中.
- 其他线程调用了notifyAll()方法。
- 其他线程通过interrupt中断了该线程。
- 等待超时时间到达，线程被自动唤醒。

刚刚上文也提到了执行notify和wait，必须先获取到资源对象的锁，获取锁的方式有三种：

- 执行对象的同步方法。
- 执行对象的同步代码块。
- 对于类对象，执行类的静态同步方法。



### wait()和sleep()

首先，wait必须要在获得了当前对象的对象锁代码块中执行（可以理解为调用哪个对象的wait方法，调用的时候就要处于哪个对象的同步代码块中），而sleep则没有这个限制。

```java
public synchronized void methodName() {
    this.wait(); //可以 没问题
}

public void methodName() {
    this.wait(); //不可以，没有处于this的同步代码块中，抛出IllegalMonitorStateException
}

public void methodName2() {
    synchronized(其他对象) {
        this.wait() // 不可以，这里只获得了其他对象的锁，但是却调用了this的wait，抛出IllegalMonitorStateException
    }
}

public void methodName3() {
    synchronized(其他对象) {
        其他对象.wait() // 可以，没问题，不一定非得是this
    }
}
```

也许你注意到了，wait()必须配合synchronized使用？为什么呢？那就是wait()有个很有意思的特性，**执行wait()后的睡眠期间，会释放掉通过synchronized获得的锁，直到被唤醒的时候会重新获得锁**，而sleep()会在线程睡眠期间依然持有这个锁，也就是说，在同步代码块中用wait()睡眠后，其他线程也能够获得对象的锁。

wait()的睡眠状态被notity()或notifyAll()唤醒时，不会抛出异常，sleep()导致的睡眠只能被interrupt()方法唤醒，同时sleep()会抛出异常，若interrupt()唤醒的是由wait()导致的睡眠，wait()同样会抛出异常。

同时，由于wait()会释放掉锁，因此一个对象中可能会同时存在多个线程由于wait()导致的睡眠

### notify()与notifyAll()

notify()会将对象的一个线程从wait()导致的睡眠状态中唤醒，至于是哪个线程，则是随机的。

而notifyAll()则是唤醒对象所有由于wait()导致的睡眠。由于wait()被唤醒后会对对象加锁，如果没能拿到锁会暂时先阻塞着直到有机会拿到锁，因此依然是线程安全的。

### 简易消息队列的实现

现在就利用上面的特性，实现一个简单的阻塞型消息队列吧。

消息队列是一个典型的消费者-生产者模型，当消费者消费时，若消息队列为空，则一直等待（阻塞执行），直到生产者生产了一个消息，此时消费者阻塞消除并获得一个消息

这里阻塞的关键是队列空时等待，非空时唤醒

消息队列类完整代码如下：

```java
/**
 * 使用锁实现的消息队列
 */
class MessageQueue {
    // 队列关闭标志
    private volatile boolean stop = false;
    // 使用链表作为消息容器
    private final LinkedList<String> queue = new LinkedList<>();
    /**
     * 生产消息
     * @param msg   消息内容
     */
    public void product(String msg) {
        // 获取链表的锁之后，向链表添加消息，最后尝试唤醒链表对象（如果在等待的话）
        synchronized (this) {
            if (stop) {
                throw new IllegalStateException("消息队列已关闭");
            }
            queue.add(msg);
            this.notify();
        }
    }
    /**
     * 关闭消息队列，并唤醒消费者的等待（如果在等待的话）
     */
    public void stop() {
        synchronized (this) {
            stop = true;
            this.notify();
        }
    }
    /**
     * 消费消息，若无消息将会一直阻塞，直到有消息或队列关闭，若消息队列已关闭则不再阻塞且返回null
     * @return 拿到的消息
     */
    public String consume() {
        // 获取对象的锁
        synchronized (this) {
            if (queue.size() != 0) {
                return queue.remove();
            } else {
                while (!stop && queue.size() == 0) {
                    try {
                        this.wait();
                    } catch (InterruptedException ignored) { }
                }

                // 由于关闭队列也有可能退出上面的循环，所以要再判断一次长度
                if (queue.size() == 0) {
                    return null;
                } else {
                    return queue.remove();
                }
            }
        }
    }
}
```

测试代码如下

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        MessageQueue mq = new MessageQueue();

        // 开1个线程消费消息
        new Thread(() -> {
           while (true) {
               String msg = mq.consume();
               System.out.println("消费消息：" + msg);
               if (msg == null) {
                   System.out.println("队列关闭");
                   break;
               }
           }
        }).start();
        for (int i = 0; i < 10; i++) {
            mq.product("" + i);
            System.out.println("生产消息：" + i);
            Thread.sleep(100);
        }
        mq.stop();
        Thread.sleep(1000);
        System.out.println("主线程结束");
    }
}
```

运行可以观察到每0.1秒生产一个消息，紧接着就被消费，最后程序退出 

