---
title: JDK-线程不安全案例分析
date: 2021-04-07 09:07:02
tags:
---

## StringBuilder类线程不安全

### 不安全的使用

```java
@Safe
public class StringBuilderDemo {  
    public static void main(String[] args) throws InterruptedException {  
        StringBuilder stringBuilder = new StringBuilder();  
        for (int i = 0; i < 10; i++){  
            new Thread(()->{
              for (int j = 0; j < 1000; j++){  
                stringBuilder.append("a");  
              }   
            }).start();  
        }  
  
        Thread.sleep(100);  
        System.out.println(stringBuilder.length());  
    }  
  
}  
```

我们看到输出了“9326”，小于预期的10000，并且还抛出了一个ArrayIndexOutOfBoundsException异常（异常不是必现）。

### 分析

StringBuilder和StringBuffer的内部实现跟String类一样，都是通过一个char数组存储字符串的，不同的是String类里面的char数组是final修饰的，是不可变的 <!--线程安全-->；而StringBuilder和StringBuffer的char数组是可变的。

```java
//父类AbstractStringBuilder
abstract class AbstractStringBuilder implements Appendable, CharSequence {
   
    char[] value;

    int count;
  
    public AbstractStringBuilder append(String str) {  
      if (str == null)  
        return appendNull();  
      int len = str.length();  
      ensureCapacityInternal(count + len);  
      str.getChars(0, len, value, count);  
      count += len;  
      return this;  
    }
}
```

#### 返回结果小于预期的原因

<font color="red">注意：count +=len不是一个原子操作！！！线程不安全，线程读取的count是失效数据，属于“先读取count，后更新count”的错误</font>

这就是为什么["我们看到输出了“9326”，小于预期的10000"]()



#### 抛出异常原因

##### ensureCapacityInternal()方法

检查StringBuilder对象的原char数组的容量是否充足，如果不充足调用expandCapacity()方法对char数组进行扩容。

```java
private void ensureCapacityInternal(int minimumCapacity) {  
        // overflow-conscious code  
    if (minimumCapacity - value.length > 0)  
        expandCapacity(minimumCapacity);  
}  
```

##### expandCapacity()方法

扩容的逻辑就是new一个2倍容量的新char数组，再通过System.arryCopy()函数将原数组的内容复制到新数组，最后将指针指向新的char数组。

```java
void expandCapacity(int minimumCapacity) {  
    //计算新的容量  
    int newCapacity = value.length * 2 + 2;  
    //中间省略了一些检查逻辑  
    ...  
    value = Arrays.copyOf(value, newCapacity);  
}  
```

##### Arrys.copyOf()方法

```java
public static char[] copyOf(char[] original, int newLength) {  
    char[] copy = new char[newLength];  
    //拷贝数组  
    System.arraycopy(original, 0, copy, 0, Math.min(original.length, newLength));  
    return copy;  
}  
```

##### str.getChars()方法

是将String对象里面char数组里面的内容拷贝到StringBuilder对象的char数组里面，代码如下：

```
str.getChars(0, len, value, count);  
```

##### getChars()方法

```java
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {  
    //中间省略了一些检查
    ...     
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);  
}  
```

拷贝流程见下图

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueq8rlWFnejuWibbkDsLW8SfkgV2icp12NDicCaAd0xklug4S51nyQCLicn9Lo9KospQpaKTfxmgAzEmQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

假设现在有两个线程同时执行了StringBuilder的append()方法，两个线程都执行完了第五行的ensureCapacityInternal()方法，此刻count=5。

这个时候线程1的cpu时间片用完了，线程2继续执行。线程2执行完整个append()方法后count变成6了

线程1继续执行第六行的str.getChars()方法的时候拿到的count值就是6了，执行char数组拷贝的时候就会抛出ArrayIndexOutOfBoundsException异常。



### 改造为线程安全

StringBuffer的append()方法都使用synchronized关键词修饰了，阻塞其他线程。

<!--提问：是否可以对类的count变量进行volatile修饰，从而保障线程安全呢？-->

回答：不可以。volatile只是保障每个线程拿到的count都是互相可见的，但是count+=使用了中间变量，中间变量是过期的

```java
    public static volatile Integer i = new Integer(0);
    public static void main(String[] args) throws InterruptedException {

        final Integer[] a = {i};
        for (int i = 0; i < 10; i++){
            new Thread(()->{
                for (int j = 0; j < 10000; j++){
                    a[0]++;
                }
            }).start();
        }

        Thread.sleep(100);
        System.out.println(a[0]);
    }
```



## HashMap线程不安全

### 不安全的使用

### 分析

#### resize方法扩容 

```java
   void addEntry(int hash, K key, V value, int bucketIndex) {  
Entry<K,V> e = table[bucketIndex];  
       table[bucketIndex] = new Entry<K,V>(hash, key, value, e);  
       if (size++ >= threshold)  
           resize(2 * table.length);  
   }  
  
   /** 
    * resize()方法如下，重要的是transfer方法，把旧表中的元素添加到新表中
    */  
   void resize(int newCapacity) {  
       Entry[] oldTable = table;  
       int oldCapacity = oldTable.length;  
       if (oldCapacity == MAXIMUM_CAPACITY) {  
           threshold = Integer.MAX_VALUE;  
           return;  
       }  
  
       Entry[] newTable = new Entry[newCapacity];  
       transfer(newTable);  
       this.table = newTable;  
       threshold = (int)(newCapacity * loadFactor);  
   }  
```

<!--我在看到resize()中的newTable是在方法中被new的，那么就是说newTable的句柄是当前线程私有的，但是newTable却赋予了线程共有的this.table，所以我怀疑各个线程在执行resize过程中会交替赋值给this.table，有线程安全问题-->

#### transfer方法复制

```java
void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : this.table) {
 
            while(null != e) {
                Entry<K,V> next = e.next;            ---------------------(1)
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity); 
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } // while
 
        }
    }
```

<!--我看到for循环这更怀疑了，一个可能被交替赋值的table变量可以被无阻塞遍历，肯定会有问题-->

### 复现

```
Map<Integer> map = new HashMap<Integer>(2); 
// 只能放置两个元素，其中的threshold为1（表中只填充一个元素时），即插入元素为1时就扩容（由addEntry方法中得知）
//放置2个元素 3 和 7，若要再放置元素8（经hash映射后不等于1）时，会引起扩容*
```

假设放置结果图如下：

<img src="https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhpORt79k2Xjw9lfLfpaEMyayjAgqSwlIbWQmFWo49679E9eaIfbBA8XbB5lezKWHiapRm5ETKiarrEQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom: 50%;" />

现在有两个线程A和B，都要执行put操作，即向表中添加元素，即线程A和线程B都会看到上面图的状态快照



##### 执行一：

 线程A执行到transfer函数中（1）处挂起（transfer函数代码中有标注）。此时在线程A的栈中, 当前的 e = 3, next = 7

##### 执行二：

线程B执行 transfer函数中的while循环，即会把原来的table变成另一个newtable（线程B自己的栈中），再写入到内存中。如下图（假设两个元素在新的hash函数下也会映射到同一个位置）

<img src="https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhpORt79k2Xjw9lfLfpaEMyaeadia5Ot4Hdribw9G8hUicTmiaZ0crEiaAvT8qAsBChMib07rBYHvqJKFialw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom:50%;" />

##### 执行三：

线程A解挂，接着执行（看到的仍是旧表），即从transfer代码（1）处接着执行，当前的 e = 3, next = 7, 上面已经描述。

处理元素 3 ， 将 3 放入 线程A自己栈的newtable中，处理3后的图如下：

<img src="https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhpORt79k2Xjw9lfLfpaEMyakr99ePzoWDjVjPmPBRliawVx9iaUsRk701RpYOs7zN3B6YRLzNLVEXFA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom:50%;" />

线程A再复制元素 7 ，当前 e = 7 ,而next值由于线程 B 修改了它的引用，所以next 为 3 ，处理后的新表如下图

<img src="https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhpORt79k2Xjw9lfLfpaEMyaW5v0XSY8pQV1BWaQh64N4s6P0OKyRPIKp3GaHWvInicayqegp5qxCZQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom:50%;" />

由于上面取到的next = 3, 接着while循环，即当前处理的结点为3， next就为null ，退出while循环，执行完while循环后，新表中的内容如下图：

![Image](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhpORt79k2Xjw9lfLfpaEMyaDc6BmPWluwcOWj88ZEAkkXIOFiaplMKYA8PRwWcbuxcqicmr9gm3d5Sw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

当操作完成，执行查找时，会陷入死循环！

### 改造为线程安全

ConcurrentHashMap对于put()中的核心代码加了synchronize