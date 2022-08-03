---
title: JDK-util-HashMap
date: 2022-04-19 09:07:43
tags:
---

# [HashMap](https://mp.weixin.qq.com/s/UOr9BOWrv67d8l1VQxqUkA)

我们大脑中首先要有这样一幅图：

![Image](https://mmbiz.qpic.cn/mmbiz_png/R7PtjL3tdAib0uwiarfrxiaEt9lmHOAhYdibMJVazadOLIHm8dB5Us2Nq4WlibbqZL4NMBNIMsRP3NibcOYT3uU7wNrw/640)



这张图囊括了HashMap中最基础的几个点：

1. `Java`中`HashMap`的实现的基础数据结构是数组，每一对`key`->`value`的键值对组成`Entity`类以双向链表的形式存放到这个数组中
2. 元素在数组中的位置由`key.hashCode()`的值决定，如果两个`key`的哈希值相等，即发生了哈希碰撞，则这两个`key`对应的`Entity`将以链表的形式存放在数组中
3. 调用`HashMap.get()`的时候会首先计算`key`的值，继而在数组中找到`key`对应的位置，然后遍历该位置上的链表找相应的值。

当然这张图中没有体现出来的有两点：

1. 为了提升整个`HashMap`的读取效率，当`HashMap`中存储的元素大小等于桶数组大小乘以负载因子的时候整个`HashMap`就要扩容，以减小哈希碰撞，具体细节我们在后文中讲代码会说到
2. 在`Java 8`中如果**桶数组**的同一个位置上的链表数量超过一个定值，则整个链表有一定概率会转为一棵红黑树。

整体来看，整个`HashMap`中最重要的点有四个：**初始化**，**数据寻址-`hash`方法**，**数据存储-`put`方法**,**扩容-`resize`方法**，只要理解了这四个点的原理和调用时机，也就理解了整个`HashMap`的设计。

## 把书读厚

如果对其中的某几个问题还有疑惑，那就说明我们还需要深入代码，把书读厚。

1. `HashMap`内部的`bucket`数组长度为什么一直都是2的整数次幂
2. `HashMap`默认的`bucket`数组是多大
3. `HashMap`什么时候开辟`bucket`数组占用内存
4. `HashMap`何时扩容？
5. 桶中的元素链表何时转换为红黑树，什么时候转回链表，为什么要这么设计？
6. `Java 8`中为什么要引进红黑树，是为了解决什么场景的问题？
7. `HashMap`如何处理`key`为`null`的键值对？

## `new HashMap()`

在`JDK 8`中，在调用`new HashMap()`的时候并没有分配数组堆内存，只是做了一些参数校验，初始化了一些常量

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

`tableSizeFor`的作用是找到大于`cap`的最小的2的整数幂，我们假设n(注意是n，不是cap哈)对应的二进制为000001xxxxxx，其中x代表的二进制位是0是1我们不关心 <!--我个人看法，使用位运算时，一定要注意最高位，最高位是符号位，不能移动，所以32bit的int，只能用到倒数第2的高位bit，所以HashMap的最大容量是2^30 -->

`n |= n >>> 1;`执行后`n`的值为：

<img src="https://mmbiz.qpic.cn/mmbiz_png/R7PtjL3tdAib0uwiarfrxiaEt9lmHOAhYdibnWhteLvazicGAkd7go3CeiabRjYN0ib1Wb5h1B8TuPOHBT1cr1K0GCaSA/640" style="zoom:33%;" />

可以看到此时`n`的二进制最高两位已经变成了1（1和0或1异或都是1），再接着执行第二行代码：

<img src="https://mmbiz.qpic.cn/mmbiz_png/R7PtjL3tdAib0uwiarfrxiaEt9lmHOAhYdibibEwy9YFEA0Gy21LJYNColicAxpW11teDQpRZvE0HqcTC1QYJ6Z7fWBQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom:33%;" />

可见`n`的二进制最高四位已经变成了1，等到执行完代码`n |= n >>> 16;`之后，`n`的二进制最低位全都变成了1，<!--就是为了创建最低位都是1的整数--> 也就是`n = 2^x - 1`其中x和n的值有关，如果没有超过`MAXIMUM_CAPACITY`，最后会返回一个2的正整数次幂，因此`tableSizeFor()`的作用就是保证返回一个比入参大的最小的2的正整数次幂。<!--说白了，就是把bit是1的最高位以后的低位，全部置为1，这就是“最小的2的正整数次幂”-->

在`JDK 7`中初始化的代码大体一致，在`HashMap`第一次`put`的时候会调用`inflateTable`计算桶数组的长度，但其算法并没有变：

```
// 第一次put时，初始化table
private void inflateTable(int toSize) {
    // Find an power of 2 >= toSize
    int capacity = roundUpToPowerOf2(toSize);
    threshold = (int)Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry(capacity);
    initHashSeedAsNeeded(capacity);
}
```

这里我们也回答了开头提出来的问题：

`HashMap`什么时候开辟`bucket`数组占用内存？答案是在`HashMap`第一次`put`的时候，无论`Java 8`还是`Java 7`都是这样实现的 <!--计算机领域，只有对象真正被使用的时候，才被初始化。类似“延迟加载”-->。这里我们可以看到两个版本的实现中，桶数组的大小都是2的正整数幂，至于为什么这么设计，看完后文你就明白了。

## `hash`

在`HashMap`这个特殊的数据结构中，`hash`函数承担着寻址定址的作用，其性能对整个`HashMap`的性能影响巨大，那什么才是一个好的`hash`函数呢？

- 计算出来的哈希值足够散列，能够有效减少哈希碰撞
- 本身能够快速计算得出，因为`HashMap`每次调用`get`和`put`的时候都会调用`hash`方法

下面是`Java 8`中的实现：

```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这里比较重要的是`(h = key.hashCode()) ^ (h >>> 16)`，这个位运算其实是将`key.hashCode()`计算出来的`hash`值的高16位与低16位继续异或，为什么要这么做呢？

我们知道`hash`函数的作用是用来确定`key`在桶数组中的位置的，在`JDK`中为了更好的性能，通常会这样写：

```
index =(table.length - 1) & key.hash();
```

回忆前文中的内容，`table.length`是一个2的正整数次幂，类似于`000100000`，这样的值减一就成了`000011111`，通过位运算可以高效寻址，这也回答了前文中提到的一个问题，`HashMap`内部的`bucket`数组长度为什么一直都是2的整数次幂？好处之一就是可以通过构造位运算快速寻址定址。

回到本小节的议题，既然计算出来的哈希值都要与`table.length - 1`做与运算，那就意味着计算出来的`hash`值只有低位有效，这样会加大碰撞几率，因此让高16位与低16位做异或，让低位保留部分高位信息，减少哈希碰撞。

我们再看`Java 7`中对hash的实现：

```
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by 
    // constant multiples at each bit position have a bounded 
    // number of collisions (approximately 8 at default load factor). 
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

`Java 7`中为了避免`hash`值的高位信息丢失，做了更加复杂的异或运算，但是基本出发点都是一样的，都是让哈希值的低位保留部分高位信息，减少哈希碰撞。

## `put`

在`Java 8`中`put`这个方法的思路分为以下几步：

1. 调用`key`的`hashCode`方法计算哈希值，并据此计算出数组下标index
2. 如果发现当前的桶数组为`null`，则调用`resize()`方法进行初始化
3. 如果没有发生哈希碰撞，则直接放到对应的桶中
4. 如果发生哈希碰撞，且节点已经存在，就替换掉相应的`value`
5. 如果发生哈希碰撞，且桶中存放的是树状结构，则挂载到树上
6. 如果碰撞后为链表，添加到链表尾，如果链表长度超过`TREEIFY_THRESHOLD`默认是8，则将链表转换为树结构
7. 数据`put`完成后，如果`HashMap`的总数超过`threshold`就要`resize`

具体代码以及注释如下：

```java
public V put(K key, V value) {
    // 调用上文我们已经分析过的hash方法
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        // 第一次put时，会调用resize进行桶数组初始化
        n = (tab = resize()).length;
    // 根据数组长度和哈希值相与来寻址，原理上文也分析过
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 如果没有哈希碰撞，直接放到桶中
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 哈希碰撞，且节点已存在，直接替换
            e = p;
        else if (p instanceof TreeNode)
            // 哈希碰撞，树结构
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 哈希碰撞，链表结构
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 链表过长，转换为树结构
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 如果节点已存在，则跳出循环
                    break;
                // 否则，指针后移，继续后循环
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            // 对应着上文中节点已存在，跳出循环的分支
            // 直接替换
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        // 如果超过阈值，还需要扩容
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

相比之下`Java 7`中的`put`方法就简单不少

```
public V put(K key, V value) {
    // 如果 key 为 null，调用 putForNullKey 方法进行处理  
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key.hashCode());
    int i = indexFor(hash, table.length);
    for (Entry<K, V> e = table[i]; e != null; e = e.next) {
        Object k;  
        if (e.hash == hash && ((k = e.key) == key
                || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(hash, key, value, i);
    return null;
}

void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K, V> e = table[bucketIndex];     // ①  
    table[bucketIndex] = new Entry<K, V>(hash, key, value, e);
    if (size++ >= threshold)
        resize(2 * table.length);    // ②  
}
```

这里有一个小细节，`HashMap`允许`put`key为`null`的键值对，但是这样的键值对都放到了桶数组的第0个桶中。

## `resize()`

`resize`是整个`HashMap`中最复杂的一个模块，如果在`put`数据之后超过了`threshold`的值，则需要扩容，扩容意味着桶数组大小变化，我们在前文中分析过，`HashMap`寻址是通过`index =(table.length - 1) & key.hash();`来计算的，现在`table.length`发生了变化，势必会导致部分`key`的位置也发生了变化，`HashMap`是如何设计的呢？

这里就涉及到桶数组长度为2的正整数幂的第二个优势了：当桶数组长度为2的正整数幂时，如果桶发生扩容（长度翻倍），则桶中的元素大概只有一半需要切换到新的桶中，另一半留在原先的桶中就可以，并且这个概率可以看做是均等的。

![Image](https://mmbiz.qpic.cn/mmbiz_png/R7PtjL3tdAib0uwiarfrxiaEt9lmHOAhYdiblzYia6ic0unz6yDBBUz9zaYTfnYCdtazFW4ibtEf8bs5F6K2zdNPK7n9w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

通过这个分析可以看到如果在即将扩容的那个位上`key.hash()`的二进制值为0，则扩容后在桶中的地址不变，否则，扩容后的最高位变为了1，新的地址也可以快速计算出来`newIndex = oldCap + oldIndex;` <!--对以前的地址完美兼容，这就是size是2的次幂的优势-->

下面是`Java 8`中的实现：

```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 如果oldCap > 0则对应的是扩容而不是初始化
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 没有超过最大值，就扩大为原先的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        // 如果oldCap为0， 但是oldThr不为0，则代表的是table还未进行过初始化
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        // 如果到这里newThr还未计算，比如初始化时，则根据容量计算出新的阈值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            // 遍历之前的桶数组，对其值重新散列
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 如果原先的桶中只有一个元素，则直接放置到新的桶中
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 如果原先的桶中是链表
                    Node<K,V> loHead = null, loTail = null;
                    // hiHead和hiTail代表元素在新的桶中和旧的桶中的位置不一致
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        // loHead和loTail代表元素在新的桶中和旧的桶中的位置一致
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 新的桶中的位置 = 旧的桶中的位置 + oldCap， 详细分析见前文
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

`Java 7`中的`resize`方法相对简单许多：

1. 基本的校验之后`new`一个新的桶数组，大小为指定入参
2. 桶内的元素根据新的桶数组长度确定新的位置，放置到新的桶数组中

```
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    boolean oldAltHashing = useAltHashing;
    useAltHashing |= sun.misc.VM.isBooted() &&
            (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
    boolean rehash = oldAltHashing ^ useAltHashing;
    transfer(newTable, rehash);
    table = newTable;
    threshold = (int) Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K, V> e : table) {
        //链表跟table[i]断裂遍历，头部往后遍历插入到newTable中
        while (null != e) {
            Entry<K, V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

## 总结

在看完了`HashMap`在`Java 8`和`Java 7`的实现之后我们回答一下前文中提出来的那几个问题：

1. `HashMap`内部的`bucket`数组长度为什么一直都是2的整数次幂

   答：这样做有两个好处，第一，可以通过`(table.length - 1) & key.hash()`这样的位运算快速寻址，第二，在`HashMap`扩容的时候可以保证同一个桶中的元素均匀的散列到新的桶中，具体一点就是同一个桶中的元素在扩容后一般留在原先的桶中，一般放到了新的桶中。

2. `HashMap`默认的`bucket`数组是多大

   答：默认是16，即时指定的大小不是2的整数次幂，`HashMap`也会找到一个最近的2的整数次幂来初始化桶数组。<!--长度不超过2^30-->

3. `HashMap`什么时候开辟`bucket`数组占用内存

   答：在第一次`put`的时候调用`resize`方法

4. `HashMap`何时扩容？

   答：当`HashMap`中的元素熟练超过阈值时，阈值计算方式是`capacity * loadFactor`，在`HashMap`中`loadFactor`是0.75

5. 桶中的元素链表何时转换为红黑树，什么时候转回链表，为什么要这么设计？

   答：当同一个桶中的元素数量大于等于8的时候元素中的链表转换为红黑树，反之，当桶中的元素数量小于等于6的时候又会转为链表，这样做的原因是避免红黑树和链表之间频繁转换，引起性能损耗

6. `Java 8`中为什么要引进红黑树，是为了解决什么场景的问题？

   答：引入红黑树是为了避免`hash`性能急剧下降，引起`HashMap`的读写性能急剧下降的场景，正常情况下，一般是不会用到红黑树的，在一些极端场景下，假如客户端实现了一个性能拙劣的`hashCode`方法，可以保证`HashMap`的读写复杂度不会低于O(lgN)

   ```
   public int hashCode() {
       return 1;
   }
   ```

7. `HashMap`如何处理`key`为`null`的键值对？

   答：放置在桶数组中下标为0的桶中

# Java 8 ConcurrentHashMap源码中隐藏2个BUG

`Java 7`的`ConcurrenHashMap`的[源码](http://mp.weixin.qq.com/s?__biz=Mzk0NjExMjU3Mg==&mid=2247484661&idx=1&sn=0005cd08cf76c6bab5727ec532983415&chksm=c30a55a6f47ddcb04891349eef82f61830c6df7495944184fd98f1253913edd9e3187e73ddae&scene=21#wechat_redirect)我建议大家都看看，那个版本的源码就是`Java`多线程编程的教科书。在`Java 7`的源码中，作者对悲观锁的使用非常谨慎，大多都转换为自旋锁加`volatile`获得相同的语义，即使最后迫不得已要用，作者也会通过各种技巧减少锁的临界区。在上一篇文章中我们也有讲到，自旋锁在临界区比较小的时候是一个较优的选择是因为它避免了线程由于阻塞而切换上下文，但本质上它也是个锁，在自旋等待期间只有一个线程能进入临界区，其他线程只会自旋消耗`CPU`的时间片。`Java 8`中`ConcurrentHashMap`的实现通过一些巧妙的设计和技巧，避开了自旋锁的局限，提供了更高的并发性能。如果说`Java 7`版本的源码是在教我们如何将悲观锁转换为自旋锁，那么在`Java 8`中我们甚至可以看到如何将自旋锁转换为无锁的方法和技巧。

## 把书读薄

![Image](https://mmbiz.qpic.cn/mmbiz_png/R7PtjL3tdA8G9gcTVIgP3JZWCv0UibUvicIeiaDmgzDGzAh50zu8uibcTzY7acGCJI84Y9J86iaJ0aPK0mD9ASTdR6Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)image

图片来源：https://www.zhenchao.org/2019/01/31/java/cas-based-concurrent-hashmap/

在开始本文之前，大家首先在心里还是要有这样的一张图，如果有同学对`HashMap`比较熟悉，那这张图也应该不会陌生。事实上在整体的数据结构的设计上`Java 8`的`ConcurrentHashMap`和`HashMap`基本上是一致的。

`Java 7`中`ConcurrentHashMap`为了提升性能使用了很多的编程技巧，但是引入`Segment`的设计还是有很大的改进空间的，`Java 7`中`ConcurrrentHashMap`的设计有下面这几个可以改进的点：

1. `Segment`在扩容的时候非扩容线程对本`Segment`的写操作时都要挂起等待的
2. 对`ConcurrentHashMap`的读操作需要做两次哈希寻址，在读多写少的情况下其实是有额外的性能损失的
3. 尽管`size()`方法的实现中先尝试无锁读，但是如果在这个过程中有别的线程做写入操作，那调用`size()`的这个线程就会给整个`ConcurrentHashMap`加锁，这是整个`ConcurrrentHashMap`唯一一个全局锁，这点对底层的组件来说还是有性能隐患的
4. 极端情况下（比如客户端实现了一个性能很差的哈希函数）`get()`方法的复杂度会退化到`O(n)`。

针对1和2，在`Java 8`的设计是废弃了`Segment`的使用，将悲观锁的粒度降低至桶维度，因此调用`get`的时候也不需要再做两次哈希了。`size()`的设计是`Java 8`版本中最大的亮点，我们在后面的文章中会详细说明。至于红黑树，这篇文章仍然不做过多阐述。接下来的篇幅会深挖细节，把书读厚，涉及到的模块有：初始化，`put`方法, 扩容方法`transfer`以及`size()`方法，而其他模块，比如`hash`函数等改变较小，故不再深究。

## 准备知识

### `ForwardingNode`

```
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        // MOVED = -1，ForwardingNode的哈希值为-1
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
}
```

除了普通的`Node`和`TreeNode`之外，`ConcurrentHashMap`还引入了一个新的数据类型`ForwardingNode`，我们这里只展示他的构造方法，`ForwardingNode`的作用有两个：

- 在动态扩容的过程中标志某个桶已经被复制到了新的桶数组中
- 如果在动态扩容的时候有`get`方法的调用，则`ForwardingNode`将会把请求转发到新的桶数组中，以避免阻塞`get`方法的调用，`ForwardingNode`在构造的时候会将扩容后的桶数组`nextTable`保存下来。

### `UNSAFE.compareAndSwap***`

这是在`Java 8`版本的`ConcurrentHashMap`实现`CAS`的工具，以`int`类型为例其方法定义如下：

```
/**
* Atomically update Java variable to <tt>x</tt> if it is currently
* holding <tt>expected</tt>.
* @return <tt>true</tt> if successful
*/
public final native boolean compareAndSwapInt(Object o, long offset,
                                              int expected,
                                              int x);
```

相应的语义为：

> 如果对象`o`起始地址偏移量为`offset`的值等于`expected`，则将该值设为`x`，并返回`true`表明更新成功，否则返回`false`，表明`CAS`失败

## 初始化

```
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0) // 检查参数
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)
        initialCapacity = concurrencyLevel;
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size); // tableSizeFor，求不小于size的 2^n的算法，jdk1.8的HashMap中说过
    this.sizeCtl = cap; 
}
```

即使是最复杂的一个初始化方法代码也是比较简单的，这里我们只需要注意两个点：

- `concurrencyLevel`在`Java 7`中是`Segment`数组的长度，由于在`Java 8`中已经废弃了`Segment`，因此`concurrencyLevel`只是一个保留字段，无实际意义
- `sizeCtl`这个值第一次出现，这个值如果等于-1则表明系统正在初始化，如果是其他负数则表明系统正在扩容，在扩容时`sizeCtl`二进制的低十六位等于扩容的线程数加一，高十六位（除符号位之外）包含桶数组的大小信息

## `put`方法

```
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

`put`方法将调用转发到`putVal`方法：

```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 【A】延迟初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 【B】当前桶是空的，直接更新
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                            new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 【C】如果当前的桶的第一个元素是一个ForwardingNode节点，则该线程尝试加入扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // 【D】否则遍历桶内的链表或树，并插入
        else {
            // 暂时折叠起来，后面详细看
        }
    }
    // 【F】流程走到此处，说明已经put成功，map的记录总数加一
    addCount(1L, binCount);
    return null;
}
```

从整个代码结构上来看流程还是比较清楚的，我用括号加字母的方式标注了几个非常重要的步骤，`put`方法依然牵扯出很多的知识点

### 桶数组的初始化

```
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            // 说明已经有线程在初始化了，本线程开始自旋
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // CAS保证只有一个线程能走到这个分支
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // sc = n - n/4 = 0.75n
                    sc = n - (n >>> 2);
                }
            } finally {
                // 恢复sizeCtl > 0相当于释放锁
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

在初始化桶数组的过程中，系统如何保证不会出现并发问题呢，关键点在于自旋锁的使用，当有多个线程都执行`initTable`方法的时候，`CAS`可以保证只有一个线程能够进入到真正的初始化分支，其他线程都是自旋等待。这段代码中我们关注三点即可：

- 依照前文所述，当有线程开始初始化桶数组时，会通过`CAS`将`sizeCtl`置为-1，其他线程以此为标志开始自旋等待
- 当桶数组初始化结束后将`sizeCtl`的值恢复为正数，其值等于0.75倍的桶数组长度，这个值的含义和之前`HashMap`中的`THRESHOLD`一致，是系统触发扩容的临界点
- 在`finally`语句中对`sizeCtl`的操作并没有使用`CAS`是因为`CAS`保证只有一个线程能够执行到这个地方

### 添加桶数组第一个元素

```
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

`put`方法的第二个分支会用`tabAt`判断当前桶是否是空的，如果是则会通过`CAS`写入，`tabAt`通过`UNSAFE`接口会拿到桶中的最新元素，`casTabAt`通过`CAS`保证不会有并发问题，如果`CAS`失败，则通过循环再进入其他分支

### 判断是否需要新增线程扩容

```
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
                (sc = sizeCtl) < 0) {
            // RESIZE_STAMP_SHIFT = 16
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 这里将sizeCtl的值自增1，表明参与扩容的线程数量+1
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

在这个地方我们就要详细说下`sizeCtl`这个标志位了，临时变量`rs`由`resizeStamp`这个方法返回

```
static final int resizeStamp(int n) {
    // RESIZE_STAMP_BITS = 16
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

因为入参`n`是一个`int`类型的值，所有`Integer.numberOfLeadingZeros(n)`的返回值介于0到32之间，如果转换成二进制

- `Integer.numberOfLeadingZeros(n)`的最大值是：00000000 00000000 00000000 00100000
- `Integer.numberOfLeadingZeros(n)`的最小值是：00000000 00000000 00000000 00000000

因此`resizeStampd`的返回值也就介于`00000000 00000000 10000000 00000000`到`00000000 00000000 10000000 00100000`之间，从这个返回值的范围可以看出来`resizeStamp`的返回值高16位全都是0，是不包含任何信息的。因此在`ConcurrrentHashMap`中，会把`resizeStamp`的返回值左移16位拼到`sizeCtl`中，这就是为什么`sizeCtl`的高16位包含整个`Map`大小的原理。有了这个分析，这段代码中比较长的`if`判断也就能看懂了

```
if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
    sc == rs + MAX_RESIZERS || transferIndex <= 0)
    break;
```

- `(sc >>> RESIZE_STAMP_SHIFT) != rs`保证所有线程要基于同一个旧的桶数组扩容
- `transferIndex <= 0`已经有线程完成扩容任务了

至于`sc == rs + 1 || sc == rs + MAX_RESIZERS`这两个判断条件如果是细心的同学一定会觉得难以理解，这个地方确实是JDK的一个`BUG`，这个`BUG`已经在`JDK 12`中修复，详细情况可以参考一下Oracle的官网：https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8214427，这两个判断条件应该写成这样：`sc == (rs << RESIZE_STAMP_SHIFT) + 1 || sc == (rs << RESIZE_STAMP_SHIFT) + MAX_RESIZERS`,因为直接比较`rs`和`sc`是没有意义的，必须要有移位操作。它表达的含义是

- `sc == (rs << RESIZE_STAMP_SHIFT) + 1`当前扩容的线程数为0，即已经扩容完成了，就不需要再新增线程扩容
- `sc == (rs << RESIZE_STAMP_SHIFT) + MAX_RESIZERS`参与扩容的线程数已经到了最大，就不需要再新增线程扩容

真正扩容的逻辑在`transfer`方法中，我们后面会详细看，不过有个小细节可以提前注意，如果`nextTable`已经初始化了，`transfer`会返回`nextTable`的的引用，后续可以直接操作新的桶数组。

### 插入新值

如果桶数组已经初始化好了，该扩容的也扩容了，并且根据哈希定位到的桶中已经有元素了，那流程就跟普通的`HashMap`一样了，唯一一点不同的就是，这时候要给当前的桶加锁，且看代码：

```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)// 折叠
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {// 折叠}
        else if ((fh = f.hash) == MOVED)// 折叠
        else {
            V oldVal = null;
            synchronized (f) {
                // 要注意这里这个不起眼的判断条件
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // fh>=0的节点是链表，否则是树节点或者ForwardingNode
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                oldVal = e.val; // 如果链表中有值了，直接更新
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                // 如果流程走到这里，则说明链表中还没值，直接连接到链表尾部
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    // 红黑树的操作先略过
                }
            }
        }
    }
    // put成功，map的元素个数+1
    addCount(1L, binCount);
    return null;
}
```

这段代码中要特备注意一个不起眼的判断条件（上下文在源码上已经标注出来了）：`tabAt(tab, i) == f`，这个判断的目的是为了处理调用`put`方法的线程和扩容线程的竞争。因为`synchronized`是阻塞锁，如果调用`put`方法的线程恰好和扩容线程同时操作同一个桶，且调用`put`方法的线程竞争锁失败，等到该线程重新获取到锁的时候，当前桶中的元素就会变成一个`ForwardingNode`，那就会出现`tabAt(tab, i) != f`的情况。

## 多线程动态扩容

```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // 初始化新的桶数组
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                        (this, TRANSFERINDEX, nextIndex,
                        nextBound = (nextIndex > stride ?
                                    nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 判断是会否是最后一个扩容线程
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED) // 只有最后一个扩容线程才有机会执行这个分支
            advance = true; // already processed
        else { // 复制过程与HashMap类似，这里不再赘述
            synchronized (f) {
               // 折叠
            }
        }
    }
}
```

在深入到源码细节之前我们先根据下图看一下在`Java 8`中`ConcurrentHashMap`扩容的几个特点：

- 新的桶数组`nextTable`是原先桶数组长度的2倍，这与之前`HashMap`一致
- 参与扩容的线程也是分段将`table`中的元素复制到新的桶数组`nextTable`中
- 桶一个桶数组中的元素在新的桶数组中均匀的分布在两个桶中，桶下标相差n(旧的桶数组的长度)，这一点依然与`HashMap`保持一致

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)image-20210424202636495

### 各个线程之间如何通力协作

先看一个关键的变量`transferIndex`，这是一个被`volatile`修饰的变量，这一点可以保证所有线程读到的一定是最新的值。

```
private transient volatile int transferIndex;
```

这个值会被第一个参与扩容的线程初始化，因为只有第一个参与扩容的线程才满足条件`nextTab == null`

```
if (nextTab == null) {            // initiating
    try {
        @SuppressWarnings("unchecked")
        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
        nextTab = nt;
    } catch (Throwable ex) {      // try to cope with OOME
        sizeCtl = Integer.MAX_VALUE;
        return;
    }
    nextTable = nextTab;
    transferIndex = n;
}
```

在了解了`transferIndex`属性的基础上，上面的这个循环就好理解了

```
while (advance) {
    int nextIndex, nextBound;
      // 当bound <= i <= transferIndex的时候i自减跳出这个循环继续干活
    if (--i >= bound || finishing)
        advance = false;
    // 扩容的所有任务已经被认领完毕，本线程结束干活
    else if ((nextIndex = transferIndex) <= 0) {
        i = -1;
        advance = false;
    }
    // 否则认领新的一段复制任务，并通过`CAS`更新transferIndex的值
    else if (U.compareAndSwapInt
                (this, TRANSFERINDEX, nextIndex,
                nextBound = (nextIndex > stride ?
                            nextIndex - stride : 0))) {
        bound = nextBound;
        i = nextIndex - 1;
        advance = false;
    }
}
```

`transferIndex`就像是一个游标，每个线程认领一段复制任务的时候都会通过CAS将其更新为`transferIndex - stride`， `CAS`可以保证`transferIndex`可以按照`stride`这个步长降到0。

### 最后一个扩容线程需要二次确认？

对于每一个扩容线程，`for`循环的变量`i`代表要复制的桶的在桶数组中的下标，这个值的上限和下限通过游标`transferIndex`和步长`stride`计算得来，当`i`减小为负数，则说明当前扩容线程完成了扩容任务，这时候流程会走到这个分支：

```
// i >= n || i + n >= nextn现在看来取不到
if (i < 0 || i >= n || i + n >= nextn) {
    int sc;
    if (finishing) { // 【A】完成整个扩容过程
        nextTable = null;
        table = nextTab;
        sizeCtl = (n << 1) - (n >>> 1); 
        return;
    }
    // 【B】判断是否是最后一个扩容线程，如果是，则需要重新扫描一遍桶数组，做二次确认
    if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
        // (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT 说明是最后一个扩容线程
        if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
            return;
        // 重新扫描一遍桶数组，做二次确认
        finishing = advance = true;
        i = n; // recheck before commit
    }
}
```

因为变量`finishing`被初始化为`false`，所以当线程第一次进入这个`if`分支的话，会先执行注释为【B】的这个分支，同时因为`sizeCtl`的低16位被初始化为参与扩容的线程数加一，因此，当条件`(sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT`满足时，就能证明当前线程就是最后一个扩容线程了，这这时候将`i`置为`n`重新扫描一遍桶数组，并且将`finishing`置为`true`保证当桶数组被扫描结束后能够进入注释为【A】的分支结束扩容。

这里就有一个问题，按照我们前面的分析，扩容线程能够通力协作，保证各自负责的桶数组的分段不重不漏，这里为什么还需要做二次确认么？有一个开发者在`concurrency-interest`这个邮件列表中也关于这件事咨询了`Doug Lea`(地址：http://cs.oswego.edu/pipermail/concurrency-interest/2020-July/017171.html)，他给出的回复是：

> Yes, this is a valid point; thanks. The post-scan was needed in a previous version, and could be removed. It does not trigger often enough to matter though, so is for now another minor tweak that might be included next time CHM is updated.

虽然`Doug`在邮件中的措辞用了could be, not often enough等，但也确认了最后一个扩容线程的二次检查是没有必要的。具体的复制过程与`HashMap`类似，感兴趣的读者可以翻一下[高端的面试从来不会在HashMap的红黑树上纠缠太多](http://mp.weixin.qq.com/s?__biz=Mzk0NjExMjU3Mg==&mid=2247484593&idx=1&sn=1329f371cc600813034535411c3a987f&chksm=c30a55e2f47ddcf47a6fc2fbd67be93ec4a87dd00dccbd60d9b602f591f9218cb526ac739a16&scene=21#wechat_redirect)这篇文章。

## `size()`方法

### `addCount()方法`

```
// 记录map元素总数的成员变量
private transient volatile long baseCount;
```

在`put`方法的最后，有一个`addCount`方法，因为`putVal`执行到此处说明已经成功新增了一个元素，所以`addCount`方法的作用就是维护当前`ConcurrentHashMap`的元素总数，在`ConcurrentHashMap`中有一个变量`baseCount`用来记录`map`中元素的个数，如下图所示，如果同一时刻有n个线程通过CAS同时操作`baseCount`变量，有且仅有一个线程会成功，其他线程都会陷入无休止的自旋当中，那一定会带来性能瓶颈。

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)image-20210420221407349

为了避免大量线程都在自旋等待写入`baseCount`，`ConcurrentHashMap`引入了一个辅助队列，如下图所示，现在操作`baseCount`的线程可以分散到这个辅助队列中去了，调用`size()`的时候只需要将`baseCount`和辅助队列中的数值相加即可，这样就实现了调用`size()`无需加锁。

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)image-20210420222306734

辅助队列是一个类型为`CounterCell`的数组：

```
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

可以简单理解为只是包装了一个`long`型的变量`value`，还需要解决一个问题是，对于某个具体的线程它是如何知道操作辅助队列中的哪个值呢？答案是下面的这个方法：

```
static final int getProbe() {
    return UNSAFE.getInt(Thread.currentThread(), PROBE);
}
```

`getProbe`方法会返回当前线程的一个唯一身份码，这个值是不会变的，因此可以将`getProbe`的返回值与辅助队列的长度作求余运算得到具体的下标，它的返回值可能是0，如果返回0则需要调用`ThreadLocalRandom.localInit()`初始化。`addCount`方法中有两个细节需要注意

```
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 注意这里的判断条件，是有技巧的
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            // 变量uncontended记录着这个CAS操作是否成功
            !(uncontended =
                U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        // 检查是否需要扩容，后面再详细看
    }
}
```

细节一：

首先我们要注意方法中刚进来的`if`判断条件：

```
if ((as = counterCells) != null ||
    !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
}
```

作者在这里巧妙的运用了逻辑短路，如果`(as = counterCells) != null`则后面的`CAS`是不会执行的，为什么要这么设置呢？作者有两点考虑：

1. 原因在于如果`(as = counterCells) != null`，则说明辅助队列已经初始化好了，相比于所有的线程都自旋等待`baseCount`这一个变量，让线程通过`CAS`去操作队列中的值有更大的可能性成功，因为辅助队列的最大长度为大于当前处理器个数的2的正整数幂，可以支持更大的并发
2. 如果辅助队列还没有初始化好，直到有必要的时候再去创建队列，如何判断“必要性”呢？就看对`baseCount`的`CAS`操作能否成功，如果失败，就说明当前系统的并发已经比较高了，需要队列的辅助，否则直接操作`baseCount`

细节二：

只有当辅助队列已存在，且由`ThreadLocalRandom.getProbe()`在辅助队列中确定的位置不为`null`时，才对其做`CAS`操作，这本来是一个正常的防御性判断，但是`uncontended`记录了`CAS`是否成功，如果失败，则会在`fullAddCount`中调用`ThreadLocalRandom.advanceProbe`换一个身份码调整下当前线程在辅助队列的位置，避免所有线程都在辅助队列的同一个坑位自旋等待。

### `fullAddCount()`方法

```
// See LongAdder version for explanation
// wasUncontended 记录着调用方CAS是否成功，如果失败则换一个辅助队列的元素继续CAS
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        // 【A】如果辅助队列已经创建，则直接操作辅助队列
        if ((as = counterCells) != null && (n = as.length) > 0) {
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {            // Try to attach new Cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // 如果调用方CAS失败了，本轮空跑，下一个循环换下标继续操作
                wasUncontended = true;      // Continue after rehash
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            else if (counterCells != as || n >= NCPU) 
                // 如果辅助队列长度已经超过了CPU个数，本轮空跑，下一个循环换下标继续操作
                collide = false;            // At max size or stale
            else if (!collide) // 如果上一次操作失败了(CAS失败或者新建CounterCell失败)，本轮空跑，下一个循环换下标继续操作
                collide = true;
            else if (cellsBusy == 0 && // 如果连续两次操作辅助队列失败，则考虑扩容
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            // 如果上一次操作失败或者调用方CAS失败，都会走到这里，变换要操作的辅助队列下标
            h = ThreadLocalRandom.advanceProbe(h);
        }
        // 【B】如果辅助队列还未创建，则加锁创建
        else if (cellsBusy == 0 && counterCells == as &&
                    U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        // 【C】如果辅助队列创建失败(拿锁失败)，则尝试直接操作`baseCount`
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

因为`counterCells`是一个普通的数组，因此对其的写操作，包括初始化，扩容以及元素的写都需要加锁，加锁的方式是对全局变量`cellsBusy`的自旋锁。先看最外层的三个分支：

- 【B】如果辅助队列还没有创建，则加锁创建
- 【C】如果因为拿锁失败导致辅助队列创建失败，则尝试自旋写入变量`baseCount`，万一真的成功了呢
- 【A】如果辅助队列已经创建了，则直接去操作辅助队列相应的元素

注释中标注【A】的这个分支代码较多，其主要思路是如果通过`CAS`或者加锁操作辅助队列中的某个元素失败，则首先通过调用`ThreadLocalRandom.advanceProbe(h)`换一个队列中的元素继续操作，这次操作是否成功会记录在临时变量`collide`中。如果下一次操作还是失败，则说明此时的并发量比较大需要扩容了。如果辅助队列的长度已经超过了`CPU`的个数，那就不再扩容，继续换一个元素操作，因为同一时间能运行的线程数最大不会超过计算机的`CPU`个数。

在这个过程中有四个细节仍然需要注意：

细节一：

`counterCells`只是一个普通的数组，因此并不是线程安全的，所以对其写操作需要加锁保证并发安全

细节二：

加锁的时候，作者做了一个`double-check`的动作，我看有的文章将其解读为“类似于单例模式的`double-check`”，这个是不对的，作者这样做的原因我们在上一篇文章中有讲过，首先第一个检查`cellsBusy == 0`是流程往下走的基础，如果`cellsBusy == 1`则直接拿锁失败退出，调用`h = ThreadLocalRandom.advanceProbe(h);`更新`h`后重试，如果`cellsBusy == 0`校验通过，则调用`CounterCell r = new CounterCell(x);`初始化一个`CounterCell`，这样做是为了减少自旋锁的临界区的大小，以此来提升并发性能

细节三：

在加锁的时候先判断下`cellsBusy`是否为0，如果为1那直接宣告拿锁失败，为什么这么做呢？因为相比于调用`UNSAFE`的`CAS`操作，直接读取`volatile`的消耗更少，如果直接读取`cellsBusy`已经能判断出拿锁失败，那就没必要再调用耗时更多的`CAS`了

细节四：

对`cellsBusy`从0到1的更改调用了`CAS`但是从1置为0却只用了赋值操作，这是因为`CAS`可以保证能走到这条语句的只有一个线程，因此可以用赋值操作来更改`cellsBusy`的值。

### `sumCount`

前面两个方法主要是把`ConcurrentHashMap`中的元素个数分散的记录到`baseCount`和辅助队列中，调用`size()`方法的时候只需要把这些值相加即可。

```
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```