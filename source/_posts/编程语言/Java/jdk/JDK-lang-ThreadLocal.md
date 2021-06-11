---
title: JDK ThreadLocal
date: 2020-10-17 19:52:16
categories: [java,oom]
tags:
---



https://zhuanlan.zhihu.com/p/129607326



### 应用场景

1. 多线程可以通过jvm的堆，同时读写共享变量，但是会出现并发问题
2. 为了解决1中的并发问题，可以使用局部变量，因为局部变量会存储在线程中的栈帧中，但是局部变量不能被一个线程中的多个方法共享
3. 为了解决2中的多个方法不能共享局部变量问题，可以使用ThreadLocal



### 源码解读

#### Thread类

Thread类有一个类型为ThreadLocal.ThreadLocalMap的实例变量threadLocals, 每个线程有一个自己的ThreadLocalMap。

```java
ThreadLocal.ThreadLocalMap threadLocals = null;

ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

#### ThreadLocal类

- 连接了Thread和ThreadLocalMap，封装了Thread和ThreadLocalMap的复合操作

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
    		m.remove(this);
}

ThreadLocalMap getMap(Thread t) {
		return t.threadLocals;
}
```



#### ThreadLocal的内部类ThreadLocalMap

ThreadLocalMap有自己的独立实现，可以简单地将它的key视作ThreadLocal，value为代码中放入的值（实际上key并不是ThreadLocal本身，而是它的一个弱引用）。每个线程在往某个ThreadLocal里塞值的时候，都会往自己的ThreadLocalMap里存，读也是以某个ThreadLocal作为引用，在自己的map里找对应的key，从而实现了线程隔离。

##### Entry数组table

```java
        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;
```

table长度必须是2的指数



##### Entry

Entry便是ThreadLocalMap里定义的节点，它继承了WeakReference类，定义了一个类型为Object的value，用于存放塞到ThreadLocal里的值。

```java
static class Entry extends WeakReference<java.lang.ThreadLocal<?>> {
    // 往ThreadLocal里实际塞入的值
    Object value;

    Entry(java.lang.ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```



1. ThreadLocal的set(T value)和get()方法，本质上是要获取当前线程的ThreadLocalMap (其实就是该线程的 ThreadLocal[]  数组)
2. jdk使用ThreadLocal的内部类, ThreadLocalMaps对 ThreadLocal[ ]数组进行了封装
3. ThreadLocalMap内部维护了Entry[ ] 数组, Entry是ThreadLocalMap的内部类

