---
title: 解决Jar包依赖冲突
date: 2021-05-05 19:10:53
tags:
---





## Java的多线程同步运行

### 2个线程轮流打印AB->AB

#### 方法1：使用wait和notify

```java
 public static void main( String[] args )
    {
        System.out.println( "Hello World!" );
        
        final Object lockObj = new Object();
        //A是否已经执行
        AtomicBoolean aExecuted = new AtomicBoolean(false);
       
        final Thread a = new Thread(() -> {
            while (true) {
                synchronized (lockObj) {
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }


                    try {
                        Thread.sleep(500);
                        while (aExecuted.get())
                            lockObj.wait();
                        System.out.println("Hello from a!");
                        aExecuted.set(true);
                        lockObj.notifyAll();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }

                }
            }
        });

        final Thread b = new Thread(() -> {
            while (true) {
                synchronized (lockObj) {
                    try {
                        Thread.sleep(1_000);
                        while (!aExecuted.get())
                            lockObj.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                    lockObj.notifyAll();
                    System.out.println("Hello from b!");
                    aExecuted.set(false);
                }
            }
        });

        a.start();
        b.start();
    }
```

#### 方法2 ：使用ReentractLock

```java
public static void main( String[] args )
    {
        final ReentrantLock lockObj = new ReentrantLock();
        //A是否已经执行
        final AtomicBoolean aExecuted = new AtomicBoolean(false);
        final Condition aCondition = lockObj.newCondition();

        final Thread a = new Thread(() -> {
            while (true) {
                lockObj.lock();
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }


                    try {
                        Thread.sleep(500);
                        while (aExecuted.get())
                            aCondition.await();
                        System.out.println("Hello from a!");
                        aExecuted.set(true);
                        aCondition.signalAll();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }

                lockObj.unlock();
            }
        });

        final Thread b = new Thread(() -> {
            while (true) {
                lockObj.lock();
                    try {
                        Thread.sleep(1_000);
                        while (!aExecuted.get())
                            aCondition.await();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                    aCondition.signalAll();
                    System.out.println("Hello from b!");
                    aExecuted.set(false);
                lockObj.unlock();
            }
        });

        a.start();
        b.start();
    }
```

使用



## Spring问题

### spring为什么使用三级缓存而不是两级？

https://www.zhihu.com/question/445446018

```
简单来讲，就是通过第三级缓存（不是三级共同作用）的延迟初始化来达到循环依赖**一级缓存：** 存放初始化完全的 bean 实例缓存（用于查找，没有循环依赖一级缓存足够使用）

**三级缓存：** bean 在实例化之后，会放入的未初始化的 bean 工厂方法来延迟初始化
本质是调用`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#getEarlyBeanReference`方法的 bean 工厂匿名类

**二级缓存：** 延迟初始化后（本质调用 getEarlyBeanReference 包装了 原先的 bean 实例）
earlySingletonObjects.put(getEarlyBeanReference 返回的实例)
singletonFactories.remove(bean 工厂实例)
本质是防止`getEarlyBeanReference`bean 方法由于每次延迟初始化返回不同实例而导致注入的 bean 不是同一个
例如`() -> getEarlyBeanReference(beanName, mbd, bean)`会调用到构建代理的组件
`org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#getEarlyBeanReference`
如果没有二级缓存，会重复创建代理的实例的问题
```

