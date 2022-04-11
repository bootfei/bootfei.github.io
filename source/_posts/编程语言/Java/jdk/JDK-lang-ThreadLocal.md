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

Thread类维护了2个类型为ThreadLocal.ThreadLocalMap的变量, threadLocals和inheritableThreadLocals（和子线程获取父线程的数据有关）。

所以每个Thread有一个独享的ThreadLocalMap。

```java
ThreadLocal.ThreadLocalMap threadLocals = null;

ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

#### ThreadLocal类

ThreadLocal有一个内部类ThreadLocalMap;

连接了Thread和ThreadLocalMap，封装了Thread和ThreadLocalMap的复合操作

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

set和get方法，都是对ThreadLocalMap的操作

```java
  public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

#### ThreadLocal内部类 - ThreadLocalMap

ThreadLocalMap有自己的独立实现，可以简单地将它的key视作ThreadLocal，value为代码中放入的值（实际上key并不是ThreadLocal本身，而是它的一个弱引用）。每个线程在往某个ThreadLocal里塞值的时候，都会往自己的ThreadLocalMap里存，读也是以某个ThreadLocal作为引用，在自己的ThreadLocalMap里找对应的key，从而实现了线程隔离。

##### Entry数组table

ThreadLocal需要维持一个最坏2/3的负载因子，对于负载因子相信应该不会陌生，在HashMap中就有这个概念。
ThreadLocal有两个方法用于得到上一个/下一个索引，注意这里实际上是环形意义下的上一个与下一个。

**由于ThreadLocalMap使用线性探测法来解决散列冲突，所以实际上Entry[]数组在程序逻辑上是作为一个环形存在的。**
<!--参考网上资料或者TAOCP（《计算机程序设计艺术》）第三卷的6.4章节。-->

虚线表示弱引用，实线表示强引用。
![img](https://images2015.cnblogs.com/blog/584724/201705/584724-20170501020337211-761293878.png)

ThreadLocalMap维护了Entry环形数组，数组中元素Entry的逻辑上的key为某个ThreadLocal对象（实际上是指向该ThreadLocal对象的弱引用），value为代码中该线程往该ThreadLoacl变量实际塞入的值。

##### 内部类Entry

Entry便是ThreadLocalMap里定义的节点，它继承了WeakReference类，定义了一个类型为Object的value，用于存放塞到ThreadLocal里的值value。

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
2. jdk使用ThreadLocal的内部类ThreadLocalMaps，对 ThreadLocal[ ]数组进行了封装
3. ThreadLocalMap内部维护了Entry[ ] 数组, Entry是ThreadLocalMap的内部类

##### 为什么Entry要继承弱引用

因为如果这里使用普通的key-value形式来定义存储结构，实质上就会造成节点的生命周期与线程强绑定，只要线程没有销毁，那么节点在GC分析中一直处于可达状态，没办法被回收，而程序本身也无法判断是否可以清理节点。弱引用是Java中四档引用的第三档，比软引用更加弱一些，如果一个对象没有强引用链可达，那么一般活不过下一次GC。当某个ThreadLocal已经没有强引用可达，则随着它被垃圾回收，在ThreadLocalMap里对应的Entry的键值会失效，这为ThreadLocalMap本身的垃圾清理提供了便利。

##### 构造函数

```java
/**
 * 构造一个包含firstKey和firstValue的map。
 * ThreadLocalMap是懒汉模式，所以只有当至少要往里面放一个元素的时候，它才会初始化。
 */
ThreadLocalMap(java.lang.ThreadLocal<?> firstKey, Object firstValue) {
    // 初始化table数组
    table = new Entry[INITIAL_CAPACITY];
    // 用firstKey的threadLocalHashCode与初始大小16取模得到哈希值
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    // 初始化该节点
    table[i] = new Entry(firstKey, firstValue);
    // 设置节点表大小为1
    size = 1;
    // 设定扩容阈值
    setThreshold(INITIAL_CAPACITY);
}
```

这个构造函数在set和get的时候都可能会被间接调用以初始化线程的ThreadLocalMap。

##### 哈希函数（比较难）

重点看一下上面构造函数中的`int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);`这一行代码。

ThreadLocal类中有一个被final修饰的类型为int的threadLocalHashCode，它在该ThreadLocal被构造的时候就会生成，相当于一个ThreadLocal的ID，而它的值来源于

```
/*
 * 生成hash code间隙为这个魔数，可以让生成出来的值或者说ThreadLocal的ID较为均匀地分布在2的幂大小的数组中。
 */
private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

可以看出，它是在上一个被构造出的ThreadLocal的ID/threadLocalHashCode的基础上加上一个魔数0x61c88647的。这个魔数的选取与斐波那契散列有关，0x61c88647对应的十进制为1640531527。斐波那契散列的乘数可以用(long) ((1L << 31) * (Math.sqrt(5) - 1))可以得到2654435769，如果把这个值给转为带符号的int，则会得到-1640531527。换句话说
`(1L << 32) - (long) ((1L << 31) * (Math.sqrt(5) - 1))`得到的结果就是1640531527也就是0x61c88647。通过理论与实践，当我们用0x61c88647作为魔数累加为每个ThreadLocal分配各自的ID也就是threadLocalHashCode再与2的幂取模，得到的结果分布很均匀。
ThreadLocalMap使用的是**线性探测法**，均匀分布的好处在于很快就能探测到下一个临近的可用slot，从而保证效率。这就回答了上文抛出的为什么大小要为2的幂的问题。为了优化效率。

对于`& (INITIAL_CAPACITY - 1)`，相信有过算法竞赛经验或是阅读源码较多的程序员，一看就明白，对于2的幂作为模数取模，可以用&(2n-1)来替代%2n，位运算比取模效率高很多。至于为什么，因为对2^n取模，只要不是低n位对结果的贡献显然都是0，会影响结果的只能是低n位。

**可以说在ThreadLocalMap中，形如`key.threadLocalHashCode & (table.length - 1)`（其中key为一个ThreadLocal实例）这样的代码片段实质上就是在求一个ThreadLocal实例的哈希值，只是在源码实现中没有将其抽为一个公用函数。**



#####  getEntry方法

这个方法会被ThreadLocal的get方法直接调用，用于获取map中某个ThreadLocal存放的值。

```java
private Entry getEntry(ThreadLocal<?> key) {
    // 根据key这个ThreadLocal的ID来获取索引，也即哈希值
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 对应的entry存在且未失效且弱引用指向的ThreadLocal就是key，则命中返回
    if (e != null && e.get() == key) {
        return e;
    } else {
        // 因为用的是线性探测，所以往后找还是有可能能够找到目标Entry的。
        return getEntryAfterMiss(key, i, e);
    }
}

/*
 * 调用getEntry未直接命中的时候调用此方法
 */
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
   
    
    // 基于线性探测法不断向后探测直到遇到空entry。
    while (e != null) {
        ThreadLocal<?> k = e.get();
        // 找到目标
        if (k == key) {
            return e;
        }
        if (k == null) {
            // 该entry对应的ThreadLocal已经被回收，调用expungeStaleEntry来清理无效的entry
            expungeStaleEntry(i);
        } else {
            // 环形意义下往后面走
            i = nextIndex(i, len);
        }
        e = tab[i];
    }
    return null;
}

/**
 * 这个函数是ThreadLocal中核心清理函数，它做的事情很简单：
 * 就是从staleSlot开始遍历，将无效（弱引用指向对象被回收）清理，即对应entry中的value置为null，将指向这个entry的table[i]置为null，直到扫到空entry。
 * 另外，在过程中还会对非空的entry作rehash。
 * 可以说这个函数的作用就是从staleSlot开始清理连续段中的slot（断开强引用，rehash slot等）
 */
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 因为entry对应的ThreadLocal已经被回收，value设为null，显式断开强引用
    tab[staleSlot].value = null;
    // 显式设置该entry为null，以便垃圾回收
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 清理对应ThreadLocal已经被回收的entry
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            /*
             * 对于还没有被回收的情况，需要做一次rehash。
             * 
             * 如果对应的ThreadLocal的ID对len取模出来的索引h不为当前位置i，
             * 则从h向后线性探测到第一个空的slot，把当前的entry给挪过去。
             */
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                
                /*
                 * 在原代码的这里有句注释值得一提，原注释如下：
                 *
                 * Unlike Knuth 6.4 Algorithm R, we must scan until
                 * null because multiple entries could have been stale.
                 *
                 * 这段话提及了Knuth高德纳的著作TAOCP（《计算机程序设计艺术》）的6.4章节（散列）
                 * 中的R算法。R算法描述了如何从使用线性探测的散列表中删除一个元素。
                 * R算法维护了一个上次删除元素的index，当在非空连续段中扫到某个entry的哈希值取模后的索引
                 * 还没有遍历到时，会将该entry挪到index那个位置，并更新当前位置为新的index，
                 * 继续向后扫描直到遇到空的entry。
                 *
                 * ThreadLocalMap因为使用了弱引用，所以其实每个slot的状态有三种也即
                 * 有效（value未回收），无效（value已回收），空（entry==null）。
                 * 正是因为ThreadLocalMap的entry有三种状态，所以不能完全套高德纳原书的R算法。
                 *
                 * 因为expungeStaleEntry函数在扫描过程中还会对无效slot清理将之转为空slot，
                 * 如果直接套用R算法，可能会出现具有相同哈希值的entry之间断开（中间有空entry）。
                 */
                while (tab[h] != null) {
                    h = nextIndex(h, len);
                }
                tab[h] = e;
            }
        }
    }
    // 返回staleSlot之后第一个空的slot索引
    return i;
}
```

我们来回顾一下从ThreadLocal读一个值可能遇到的情况：
根据入参threadLocal的threadLocalHashCode对表容量取模得到index

- 如果index对应的slot就是要读的threadLocal，则直接返回结果
- 调用getEntryAfterMiss线性探测，过程中每碰到无效slot，调用expungeStaleEntry进行段清理；如果找到了key，则返回结果entry
- 没有找到key，返回null

##### 4.7 set方法

```
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    // 线性探测
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 找到对应的entry
        if (k == key) {
            e.value = value;
            return;
        }
        // 替换失效的entry
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold) {
        rehash();
    }
}

private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // 向前扫描，查找最前的一个无效slot
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len)) {
        if (e.get() == null) {
            slotToExpunge = i;
        }
    }

    // 向后遍历table
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 找到了key，将其与无效的slot交换
        if (k == key) {
            // 更新对应slot的value值
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            /*
             * 如果在整个扫描过程中（包括函数一开始的向前扫描与i之前的向后扫描）
             * 找到了之前的无效slot则以那个位置作为清理的起点，
             * 否则则以当前的i作为清理起点
             */
            if (slotToExpunge == staleSlot) {
                slotToExpunge = i;
            }
            // 从slotToExpunge开始做一次连续段的清理，再做一次启发式清理
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 如果当前的slot已经无效，并且向前扫描过程中没有无效slot，则更新slotToExpunge为当前位置
        if (k == null && slotToExpunge == staleSlot) {
            slotToExpunge = i;
        }
    }

    // 如果key在table中不存在，则在原地放一个即可
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 在探测过程中如果发现任何无效slot，则做一次清理（连续段清理+启发式清理）
    if (slotToExpunge != staleSlot) {
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }
}

/**
 * 启发式地清理slot,
 * i对应entry是非无效（指向的ThreadLocal没被回收，或者entry本身为空）
 * n是用于控制控制扫描次数的
 * 正常情况下如果log n次扫描没有发现无效slot，函数就结束了
 * 但是如果发现了无效的slot，将n置为table的长度len，做一次连续段的清理
 * 再从下一个空的slot开始继续扫描
 * 
 * 这个函数有两处地方会被调用，一处是插入的时候可能会被调用，另外个是在替换无效slot的时候可能会被调用，
 * 区别是前者传入的n为元素个数，后者为table的容量
 */
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        // i在任何情况下自己都不会是一个无效slot，所以从下一个开始判断
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            // 扩大扫描控制因子
            n = len;
            removed = true;
            // 清理一个连续段
            i = expungeStaleEntry(i);
        }
    } while ((n >>>= 1) != 0);
    return removed;
}


private void rehash() {
    // 做一次全量清理
    expungeStaleEntries();

    /*
     * 因为做了一次清理，所以size很可能会变小。
     * ThreadLocalMap这里的实现是调低阈值来判断是否需要扩容，
     * threshold默认为len*2/3，所以这里的threshold - threshold / 4相当于len/2
     */
    if (size >= threshold - threshold / 4) {
        resize();
    }
}

/*
 * 做一次全量清理
 */
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null) {
            /*
             * 个人觉得这里可以取返回值，如果大于j的话取了用，这样也是可行的。
             * 因为expungeStaleEntry执行过程中是把连续段内所有无效slot都清理了一遍了。
             */
            expungeStaleEntry(j);
        }
    }
}

/**
 * 扩容，因为需要保证table的容量len为2的幂，所以扩容即扩大2倍
 */
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; 
            } else {
                // 线性探测来存放Entry
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null) {
                    h = nextIndex(h, newLen);
                }
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

我们来回顾一下ThreadLocal的set方法可能会有的情况

- 探测过程中slot都不无效，并且顺利找到key所在的slot，直接替换即可
- 探测过程中发现有无效slot，调用replaceStaleEntry，效果是最终一定会把key和value放在这个slot，并且会尽可能清理无效slot
  - 在replaceStaleEntry过程中，如果找到了key，则做一个swap把它放到那个无效slot中，value置为新值
  - 在replaceStaleEntry过程中，没有找到key，直接在无效slot原地放entry
- 探测没有发现key，则在连续段末尾的后一个空位置放上entry，这也是线性探测法的一部分。放完后，做一次启发式清理，如果没清理出去key，并且当前table大小已经超过阈值了，则做一次rehash，rehash函数会调用一次全量清理slot方法也即expungeStaleEntries，如果完了之后table大小超过了threshold - threshold / 4，则进行扩容2倍

##### 4.8 remove方法

```
/**
 * 从map中删除ThreadLocal
 */
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len - 1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            // 显式断开弱引用
            e.clear();
            // 进行段清理
            expungeStaleEntry(i);
            return;
        }
    }
}
```

remove方法相对于getEntry和set方法比较简单，直接在table中找key，如果找到了，把弱引用断了做一次段清理。



#### InheritableThreadLocal原理

ThreadLocal本身是线程隔离的，InheritableThreadLocal提供了一种父子线程之间的数据共享机制。

它的具体实现是在Thread类中除了threadLocals外还有一个`inheritableThreadLocals`对象。

在线程对象初始化的时候，会调用ThreadLocal的`createInheritedMap`从父线程的`inheritableThreadLocals`中把有效的entry都拷过来
![img](https://images2015.cnblogs.com/blog/584724/201705/584724-20170520151800494-707226209.png)

可以看一下其中的具体实现

```
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                // 这里的childValue方法在InheritableThreadLocal中默认实现为返回本身值，可以被重写
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

还是比较简单的，做的事情就是以父线程的`inheritableThreadLocalMap`为数据源，过滤出有效的entry，初始化到自己的`inheritableThreadLocalMap`中。其中childValue可以被重写。

需要注意的地方是`InheritableThreadLocal`只是在子线程创建的时候会去拷一份父线程的`inheritableThreadLocals`。如果父线程是在子线程创建后再set某个InheritableThreadLocal对象的值，对子线程是不可见的。

###  常见问题1:内存泄漏

关于ThreadLocal是否会引起内存泄漏也是一个比较有争议性的问题

- 认为ThreadLocal会引起内存泄漏，因为如果一个ThreadLocal实例对象被回收了，entry.key指向null，但是entry.value对于这个链路依旧可达
  **当前线程 -> 当前线程的threadLocals(ThreadLocal.ThreadLocalMap对象）-> Entry数组 -> 某个entry.value**
  因此value不会被回收。

- 认为ThreadLocal不会引起内存泄，是因为ThreadLocal.ThreadLocalMap源码实现中自带一套自我清理的机制，对应线程之后调用ThreadLocal的get和set方法都有**很高的概率**会顺便清理掉无效对象，断开value强引用，从而大对象被收集器回收。

  

之所以有关于内存泄露的讨论是因为在有线程复用如线程池的场景中，一个线程的寿命很长，大对象value长期不被回收影响系统运行效率与安全。如果线程不会复用，用完即销毁了也不会有ThreadLocal引发内存泄露的问题。《Effective Java》一书中的第6条对这种内存泄露称为`unintentional object retention`(无意识的对象保留）。

如果在使用的ThreadLocal的过程中，显式地进行remove是个很好的编码习惯，这样是不会引起内存泄漏。



只能说如果但无论如何，我们应该考虑到何时调用ThreadLocal的remove方法。一个比较熟悉的场景就是对于一个请求一个线程的server如tomcat，在代码中对web api作一个切面，存放一些如用户名等用户信息，在连接点方法结束后，再显式调用remove。

**示例：**

由于`ThreadLocal`的`key`是弱引用，因此如果使用后不调用`remove`清理的话会导致对应的`value`内存泄露。

```
@Test
public void testThreadLocalMemoryLeaks() {
    ThreadLocal<List<Integer>> localCache = new ThreadLocal<>();
   List<Integer> cacheInstance = new ArrayList<>(10000);
    localCache.set(cacheInstance);
    localCache = new ThreadLocal<>();
}
```

当`localCache`的值被重置之后`cacheInstance`被`ThreadLocalMap`中的`value`引用，无法被GC，但是其`key`对`ThreadLocal`实例的引用是一个弱引用，本来`ThreadLocal`的实例被`localCache`和`ThreadLocalMap`的`key`同时引用，但是当`localCache`的引用被重置之后，则`ThreadLocal`的实例只有`ThreadLocalMap`的`key`这样一个弱引用了，此时这个`ThreadLocal`的实例在GC的时候能够被清理，但是`ThreadLocalMap`的`value`仍然被引用，该如何清理呢？

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/4xfJbk4AmfjWImbjAuEDxAaUmzknXeLe7yjJLic9ZC5zYGPvHm204A89nib4TrE951uEicndeibiba8OK5CibPbeJ0BQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

其实看过`ThreadLocal`源码的同学会知道，`ThreadLocal`本身对于`key`为`null`的`Entity`有自清理的过程，但是这个过程是依赖于后续对`ThreadLocal`的继续使用，假如上面的这段代码是处于一个秒杀场景下，会有一个瞬间的流量峰值，这个流量峰值也会将集群的内存打到高位(或者运气不好的话直接将集群内存打满导致故障)，后面由于峰值流量已过，对`ThreadLocal`的调用也下降，会使得`ThreadLocal`的自清理能力下降，造成内存泄露。`ThreadLocal`的自清理是锦上添花，千万不要指望他雪中送碳。

相比于`ThreadLocal`中存储的`value`对象泄露，`ThreadLocal`用在`web`容器中时更需要注意其引起的`ClassLoader`泄露。

`Tomcat`官网对在`web`容器中使用`ThreadLocal`引起的内存泄露做了一个总结，详见：https://cwiki.apache.org/confluence/display/tomcat/MemoryLeakProtection，这里我们列举其中的一个例子。

熟悉`Tomcat`的同学知道，Tomcat中的web应用由`Webapp Classloader`这个类加载器的，并且`Webapp Classloader`是破坏双亲委派机制实现的，即所有的`web`应用先由`Webapp classloader`加载，这样的好处就是可以让同一个容器中的`web`应用以及依赖隔离。

下面我们看具体的内存泄露的例子：

```
public class MyCounter {
 private int count = 0;

 public void increment() {
  count++;
 }

 public int getCount() {
  return count;
 }
}

public class MyThreadLocal extends ThreadLocal<MyCounter> {
}

public class LeakingServlet extends HttpServlet {
 private static MyThreadLocal myThreadLocal = new MyThreadLocal();

 protected void doGet(HttpServletRequest request,
   HttpServletResponse response) throws ServletException, IOException {

  MyCounter counter = myThreadLocal.get();
  if (counter == null) {
   counter = new MyCounter();
   myThreadLocal.set(counter);
  }

  response.getWriter().println(
    "The current thread served this servlet " + counter.getCount()
      + " times");
  counter.increment();
 }
}
```

需要注意这个例子中的两个非常关键的点：

- `MyCounter`以及`MyThreadLocal`必须放到`web`应用的路径中，保被`Webapp Classloader`加载
- `ThreadLocal`类一定得是`ThreadLocal`的继承类，比如例子中的`MyThreadLocal`，因为`ThreadLocal`本来被`Common Classloader`加载，其生命周期与`Tomcat`容器一致。`ThreadLocal`的继承类包括比较常见的`NamedThreadLocal`，注意不要踩坑。

假如`LeakingServlet`所在的`Web`应用启动，`MyThreadLocal`类也会被`Webapp Classloader`加载，如果此时web应用下线，而线程的生命周期未结束(比如为`LeakingServlet`提供服务的线程是一个线程池中的线程)，那会导致`myThreadLocal`的实例仍然被这个线程引用，而不能被GC，期初看来这个带来的问题也不大，因为`myThreadLocal`所引用的对象占用的内存空间不太多，问题在于`myThreadLocal`间接持有加载web应用的`webapp classloader`的引用（通过`myThreadLocal.getClass().getClassLoader()`可以引用到），而加载web应用的`webapp classloader`有持有它加载的所有类的引用，这就引起了`Classloader`泄露，它泄露的内存就非常可观了。







### 线程池中线程上下文丢失

`ThreadLocal`不能在父子线程中传递，因此最常见的做法是把父线程中的`ThreadLocal`值拷贝到子线程中，因此大家会经常看到类似下面的这段代码：

```
for(value in valueList){
     Future<?> taskResult = threadPool.submit(new BizTask(ContextHolder.get()));//提交任务，并设置拷贝Context到子线程
     results.add(taskResult);
}
for(result in results){
    result.get();//阻塞等待任务执行完成
}
```

提交的任务定义长这样：

```
class BizTask<T> implements Callable<T>  {
    private String session = null;
    
    public BizTask(String session) {
        this.session = session;
    }
    
    @Override
    public T call(){
        try {
            ContextHolder.set(this.session);
            // 执行业务逻辑
        } catch(Exception e){
            //log error
        } finally {
            ContextHolder.remove(); // 清理 ThreadLocal 的上下文，避免线程复用时context互串
        }
        return null;
    }
}
```

对应的线程上下文管理类为：

```
class ContextHolder {
    private static ThreadLocal<String> localThreadCache = new ThreadLocal<>();
    
    public static void set(String cacheValue) {
        localThreadCache.set(cacheValue);
    }
    
    public static String get() {
        return localThreadCache.get();
    }
    
    public static void remove() {
        localThreadCache.remove();
    }
    
}
```

这么写倒也没有问题，我们再看看线程池的设置：

```
ThreadPoolExecutor executorPool = new ThreadPoolExecutor(20, 40, 30, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>(40), new XXXThreadFactory(), ThreadPoolExecutor.CallerRunsPolicy);
```

其中最后一个参数控制着当线程池满时，该如何处理提交的任务，内置有4种策略

```
ThreadPoolExecutor.AbortPolicy //直接抛出异常
ThreadPoolExecutor.DiscardPolicy //丢弃当前任务
ThreadPoolExecutor.DiscardOldestPolicy //丢弃工作队列头部的任务
ThreadPoolExecutor.CallerRunsPolicy //转串行执行
```

可以看到，我们初始化线程池的时候指定如果线程池满，则新提交的任务转为串行执行，那我们之前的写法就会有问题了，串行执行的时候调用`ContextHolder.remove();`会将主线程的上下文也清理，即使后面线程池继续并行工作，传给子线程的上下文也已经是`null`了，而且这样的问题很难在预发测试的时候发现。

### 并行流中线程上下文丢失

如果`ThreadLocal`碰到并行流，也会有很多有意思的事情发生，比如有下面的代码：

```
class ParallelProcessor<T> {
    
    public void process(List<T> dataList) {
        // 先校验参数，篇幅限制先省略不写
        dataList.parallelStream().forEach(entry -> {
            doIt();
        });
    }
    
    private void doIt() {
        String session = ContextHolder.get();
        // do something
    }
}
```

这段代码很容易在线下测试的过程中发现不能按照预期工作，因为并行流底层的实现也是一个`ForkJoin`线程池，既然是线程池，那`ContextHolder.get()`可能取出来的就是一个`null`。我们顺着这个思路把代码再改一下：

```
class ParallelProcessor<T> {
    
    private String session;
    
    public ParallelProcessor(String session) {
        this.session = session;
    }
    
    public void process(List<T> dataList) {
        // 先校验参数，篇幅限制先省略不写
        dataList.parallelStream().forEach(entry -> {
            try {
                ContextHolder.set(session);
                // 业务处理
                doIt();
            } catch (Exception e) {
                // log it
            } finally {
                ContextHolder.remove();
            }
        });
    }
    
    private void doIt() {
        String session = ContextHolder.get();
        // do something
    }
}
```

修改完后的这段代码可以工作吗？如果运气好，你会发现这样改又有问题，运气不好，这段代码在线下运行良好，这段代码就顺利上线了。不久你就会发现系统中会有一些其他很诡异的bug。原因在于并行流的设计比较特殊，父线程也有可能参与到并行流线程池的调度，那如果上面的`process`方法被父线程执行，那么父线程的上下文会被清理。导致后续拷贝到子线程的上下文都为`null`，同样产生丢失上下文的问题，关于并行流的实现可以参考文章[啥？用了并行流还更慢了](http://mp.weixin.qq.com/s?__biz=MzkyMjIzOTQ3NA==&mid=2247484618&idx=1&sn=c83c0d8927dd650e25f548cd6a2b0df3&chksm=c1f62c57f681a541ebcdb4ae25d5007e73747fc8514916f3233c40c848d2b96ebc442b74b72d&scene=21#wechat_redirect)。