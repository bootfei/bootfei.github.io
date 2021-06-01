---
title: 查找CPU飙升的原因
date: 2020-11-05 09:24:42
tags: [java, jstack]
---



## 线程死锁

### 死锁代码

```java
	public static void main(String[] args){
        new Thread(()->{
            synchronized (l1){
                System.out.println("holding l1");
                Thread.sleep(500);              
                synchronized (l2){
                }
                System.out.println("releasing l1");
            }
        }).start();

        new Thread(()->{
            synchronized (l2){
                System.out.println("holding l2");
                Thread.sleep(500);
                synchronized (l1){
                }
                System.out.println("releasing l2");
            }
        }).start();

        while (true);
    }
```



### 使用jstack命令

```
jstack [pid] | grep BLOCKED
```

看到两个线程blocked，他们互相waiting to lock 和locked

```shell
"Thread-1" #13 prio=5 os_prio=31 tid=0x00007fe5aea2c800 nid=0x4003 waiting for monitor entry [0x0000700006eba000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at Solution.lambda$main$1(Solution.java:38)
	- waiting to lock <0x00000007957da970> (a java.lang.Object)
	- locked <0x00000007957da980> (a java.lang.Object)
	at Solution$$Lambda$2/2094777811.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)

"Thread-0" #12 prio=5 os_prio=31 tid=0x00007fe5aea2b800 nid=0x3d03 waiting for monitor entry [0x0000700006db7000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at Solution.lambda$main$0(Solution.java:25)
	- waiting to lock <0x00000007957da980> (a java.lang.Object)
	- locked <0x00000007957da970> (a java.lang.Object)
	at Solution$$Lambda$1/2075203460.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)
```



## IO 异常

