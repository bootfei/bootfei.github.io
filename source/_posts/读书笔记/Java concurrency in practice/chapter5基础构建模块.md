---
title: chapter5基础构建模块
date: 2020-12-19 21:07:28
tags: [并发]
---

第四章介绍了构造线程安全类时采用的一些技术，例如将线程安全性委托给现有的线程安全类。委托是创建线程安全类的一个最有效的策略，只需让现有的线程安全类管理所有的状态即可。
JAVA平台类库中包含了一个并发构建块的丰富集合，如线程安全的容器与同步工具。

## 同步容器类

同步容器类包括Vector和HashTable，此外还包括JDK1.2中才被加入的类。同步包装类Collections.synchronizedXxx工厂方法创建的。这些类实现线程安全的方式是：将它们的状态封装起来，并对每个公有方法都进行同步，使得每次只有一个线程能访问容器的状态。<!--哎，这些同步，即阻塞，实际上是避免了并发访问，降低了性能-->

### 同步容器类的问题

同步容器类都是线程安全的，但在某些情况下，可能需要额外的客户端加锁来保护复合操作。

容器上常见的复合操作包括：

- 迭代
  - **反复访问元素，直到遍历完容器中的所有元素**。
- 跳转
  - **根据指定顺序找到当前元素的下一个元素**。
- 条件运算，例如**「"若没有，则添加"」**
  - **检查在 Map 中是否存在键值 K，如果没有，就加入一个二元组 (K,V)。**

在同步容器类中，这些复合操作在没有客户端加锁的情况下仍然是线程安全的，但当其他线程并发地修改容器时，它们可能会出现意料之外的行为。

<!--同步容器的这些复合操作，本身是没有线程问题的，但是如果有多个线程对同步容器同时进行操作，那么就会造成这些复合操作互相影响对方的情况， 即并发问题。-->

操作Vector（同步容器）的复合操作可能导致混乱的结果：

```java
public static Object getLast(Vector list) {
    int lastIndex = list.size() - 1;
    return list.get(lastIndex);
}
public static void deleteLast(Vector list) {
	int lastIndex = list.size() - 1;
	list.remove(lastIndex);
}
```

这些方法看似没有任何问题，无论多少个线程同时调用它们，也不破坏Vector。如果线程A在包含10个元素的Vector 上调用getLast，同时线程B在同一个Vector 上调用deleteLast，getLast 将抛出ArrayIndexOutOfBoundsException异常。在调用 size与调用getLast这两个操作之间，Vector变小了，因此在调用size时得到的索引值将不再有效。

![img](https://img-blog.csdnimg.cn/img_convert/63543475d08cb404311697b63efe6a76.png)

由于同步容器要遵守同步策略，即支持客户端加锁，因此可能会创建一些新的操作，只要我们知道应该使用哪一个锁，那么这些新操作就与容器的其他操作一样都是原子操作。同步容器类通过其自身的锁来保护它的每个方法。通过给得容器类的锁，我们可以使getLast和deleteLast成为原子操作，并确保Vector的大小在调用size和get之间不会发生变化：

```java
public static Object getLast(Vector list) {
	synchronized (list){
		int lastIndex = list.size() - 1;
		return list.get(lastIndex);
	}
}
 
public static void deleteLast(Vector list) {
	synchronized (list){
		int lastIndex = list.size() - 1;
		list.remove(lastIndex); 
	}  
}

```

在调用size和相应的get之间，Vector的长度可能会发生变化，这种风险在对Vector中的元素进行迭代时仍然会出现，多线程环境下迭代过程中也可能抛出ArrayIndexOutOfBoundsException：

```java
for (int i = 0; i < vector.size(); i++)
    doSomething(vector.get(i));
```

<!--迭代过程中可能抛出异常，但并不意味着Vector就不是线程安全-->。Vector的状态仍然是有效的，事实上异常恰好使它保持规范的一致性。然而，在正常或迭代读过程中抛出异常的确不是人们所期望的。

造成迭代不可靠的问题同样可以通过在客户端加锁来完成，通过在迭代期间持有Vector的锁，防止其他线程在迭代期间修改Vector，这样完全阻止了其他线程在这期间访问它，如果集合很大或者对每个元素执行的任务耗时比较长，**这会削弱并发性**。

```java
synchronized (vector) {
    for (int i = 0; i < vector.size(); i++)
        doSomething(vector.get(i));//还要持有另一个锁，这是一个产生死锁风险的因素
}
```

### 迭代器与ConcurrentModificationException

尽管上面讨论的Vector是“遗留”下来的容器类，这只是说明同步容器有这样的问题。其实，“现代”的容器类也并没有消除复合操作产生的问题，比如迭代复合操作，当其他线程并发修改容器时，使用迭代器仍然避免不了在使用的地方加锁，在设计同步容器返回迭代器时，并没有使用同步（注，这里讲的是说返回的迭代器不是线程安全，而不是指返回迭代器的方法iterator() 没有使用同步，它本身就是经过同步了的。），因为他们是“及时失败”——只要有其他线程修改容器结果，立马就会抛出未检查性异常ConcurrentModificationException。

这种”及时失败“的迭代器并不是一种完备的处理机制，而只是”善意地“捕获并发错误，因此只能作为并发问题的预警指示器。它们采用的实现方式是，将计算器的变化与容器关联起来：如果在迭代期间计数器被修改，那么hasNext或next将抛出ConcurrentModificationException。然而，这种检查是在没有同步的情况下进行的，因此可能会看到失效的计数值，而迭代器可能并没有意识到已经发生了修改。这是设计上得权衡，从而降低并发修改并发操作的检测代码对程序性能带来的影响。

注：ConcurrentModificationException也可能出现在单线程的代码中，如果对象不是调用Iterator.remove，而是直接从容器中删除就会出现这种情况。

for-each循环语法对容器进行迭代时，也是隐式地用到了Iterator，从内部来看，javac将生成使用Iterator的代码，反复调用hasNext和next来迭代List对象，与迭代Vector一样，想要避免ConcurrentModificationException，就必须在迭代过程持有容器的锁：

```java
List<Widget> widgetList = Collections.synchronizedList(new ArrayList<Widget>());
...
// May throw ConcurrentModificationException
for (Widget w : widgetList)
    doSomething(w);
```

有时候开发人员并不希望在迭代期间对容器加锁，例如，某些线程在可以访问容器之前，必须等待迭代过程结束，如果容器规模很大，或者在每个元素上执行操作的时间很长，那么这些线程将长时间等待。即使不存在饥饿或者死锁等风险，长时间地对容器加锁也会降低程序的可伸缩性。持有锁的时间越长，那么在锁上的竞争就可能越激烈，如果许多线程在等待锁被释放，那么将极大地降低吞吐量和CPU的利用率。<!--这一点如同mysql的Update操作，导致数据被加上行锁（排他），其他读写操作就会一直等待甚至超时报错-->

**如果不希望在迭代期间对容器加锁，那么一种替代方法就是”克隆“容器，并在副本上进行迭代。**由于副本被封闭在线程内，因此其他线程不会在迭代期间对其进行修改，这样就避免了抛出ConcurrentModificationException（在克隆过程中仍然需要对容器加锁）。在克隆容器时存在显著地性能开销。这种方式的好坏取决于多个因素，包括容器的大小，在每个元素上执行的工作，迭代操作相对于容器其他操作的调用频率，以及在响应时间和吞吐量等方面的需求。

### 隐藏迭代器

你必须要记住，所有需要对共享容器进行迭代的地方都需要加锁。实际情况更加复杂，因为迭代器有时是隐藏的，就像下面代码一样，容器的toString方法的实现是通过迭代容器中的每个元素。编译器将字符串的连接操作转换为调用StringBuilder.append(Object),而这个方法又会调用容器的toString方法，标准容器的toString方法将迭代容器，并在每个元素上调用toString来生成容器内容的格式化表示。

```java
public class HiddenIterator {
    private final Set<Integer> set = new HashSet<Integer>();
    public synchronized void add(Integer i) { set.add(i); }
    public void addTenThings() {
        Random r = new Random();
        for (int i = 0; i < 10; i++)
            add(r.nextInt());
        System.out.println("DEBUG: added ten elements to " + set);//这里会隐式地使用迭代
   }
}
```

toString对容器进行迭代。当然真正的问题是HiddenIterator不是线程安全的。在使用println的set之前必须首先获取HiddenIterator的锁，但在调试代码和日志代码中通常会忽视这个要求。

如果状态与保护它的同步代码之间相隔越远，那么开发人员就越容易忘记在访问状态时使用正确的同步。如果将HashSet包装为synchronizedSet，并且对同步代码进行封装，就不会出现ConcurrentModificationException异常了。

> 正如封装对象的状态有助于维持不变性条件一样，封装对象的同步机制同样有助于确保实施同步策略。

容器的hashCode和equals等方法也会间接地执行迭代操作，当容器作为另一个容器的元素或键值时，就会出现这种情况。同样，containsAll、removeAll和retainAll等方法，以及把容器作为参数的构造函数，都会对容器进行迭代。所有这些间接地迭代操作都可能抛出ConcurrentModificationException。

## 并发容器

Java5.0提供了多种并发容器类来改进同步容器的性能。同步容器将所有对容器状态的访问都串行化，以实现它们的线程安全性。**代价是降低并发性，当多个线程竞争容器的锁时，吞吐量将严重减低**。<!--就是就是，锁这种依靠同步，来排斥和阻塞其他线程的设计，分明是搞串行化设计，多核系统还有什么用-->

另一方面，并发容器是针对多个线程并发访问设计的。在JAVA5.0中，增加了`ConcurrentHashMap`用来替代同步且基于散列的Map。增加了`CopyOnWriteArrayList`，用于在遍历操作为主要操作情况下代替同步的`List`。

在新的ConcurrentMap接口中增加了对一些常见复合操作的支持，例如"若没有刚添加"、替换以及有条件删除等。

> 通过并发容器来代替同步容器，可以极大地提高伸缩性并降低风险。

Java5.0增加了两种新的容器类型：`Queue`和`BlockingQueue`。

`Queue`用来保存一组等待处理的元素。它提供了几组实现：

- `ConcurrentLinkedQueue`，先进先出队列。

- `PriorityQueue`，非并发的优先队列。

`Queue`上的操作**不会阻塞**，如果队列为空，返回空。虽然可以用List来模拟Queue的行为 --- 事实上Queue是用LinkedList来实现Queue，但是**还是需要一个Queue类，因为Queue能去掉List的随机访问需求，从而实现更高效的并发**。

`BlockingQueue`**扩展了**Queue，**增加了可阻塞的插入和获取等操作**。队列为空，阻塞获取直到出现一个可用元素。如果队列满了（对于有界队列），那么插入元素的操作将一直阻塞，直到队列中出现可用的空间。<!--在生产者----消费者设计模式中，阻塞队列是非常有用的。-->

Java6引入了`ConcurrentListMap`和`ConcurrentSkipListSet`，分别作为同步的`SortedMap`和`SortedSet`的并发替代品。例如用synchronizedMap包装的TreeMap或TreeSet。

### ConcurrentHashMap

**同步容器类在执行每个操作期间都持有一个锁。**<!--先给出了一个重要的结论-->

在一些操作中，例如HashMap.get或List.contains, 可能包含大最的工作： 当遍历散列桶或链表来查找某个特定的对象时，必须在许多元素上调用equals (而equals本身还包含一定的计算量）。在基于散列的容器中， 如果`hashCode`**不能很均匀地分布散列值，那么容器中的元素就不会均匀地分布在整个容器中**。

> 某些情况下，某个糟糕的散列函数还会把一个**散列表变成线性链表**。当遍历很长的链表并且在某些或者全部元素上调用equals方法时，会花费很长的时间，而其他线程在这段时间内都不能访问该容器。
>

<!--【使用同步容器类在并发环境下的窘境，所以 JDK5的时候引入了并发包，引入了并发容器，这是并发容器为什么出现的技术背景】-->

与HashMap 一样，ConcurrentHashMap也是一个基于散列的Map, 但它使用了一种完全不同的加锁策略来提供更高的并发性和伸缩性。

**ConcurrentHashMap 并不是将每个方法都在同一个锁上同步并使得每次只能有一个线程访问容器， 而是使用一种粒度更细的加锁机制来实现更大程度的共享，这种机制称为分段锁(Lock Striping, 请参考11.4.3节）。**<!--分段锁的优势，就是支持并发，而不是同步-->

在这种机制中，任意数量的读取线程可以并发地访问Map, 执行读取操作的线程和执行写入操作的线程可以并发地访问Map, 并且一定数量的写入线程可以并发地修改Map。ConcurrentHashMap带来的结果是，**在并发访问环境下将实现更高的吞吐， 而在单线程环境中只损失非常小的性能。**

ConcurrentHashMap与其他并发容器一起增强了同步容器类：

-  它们提供的迭代器不会抛出ConcurrentModificationException, 因此**不需要在迭代过程中对容器加锁**。<!--提高了伸缩性-->
- ConcurrentHashMap返回的迭代器具有**弱一致性**(Weakly Consistent), 而并非“ 及时失败"。**弱一致性的迭代器可以容忍并发的修改，当创建迭代器时会遍历已有的元素， 并可以（但是不保证） 在迭代器被构造后将修改操作反映给容器。**

尽管有这些改进，但仍然需要一些权衡的因素。 对于一些需要在整个 Map 上进行计算的方法，例如 `size` 和 `isEmpty` ，**这些方法的语义被略微减弱了，以反映容器的并发特性**。

由于 `size`返回的结果在计算时可能已经过期了，它实际上只是一个估计值，因此允许 `size` 返回一个近似值而不是 精确值。 虽然这看上去令人有些不安，但事实上， `size` 和 `isEmpty`这样的方法在并发环境下的用处很小，因为它们的返回值总在不断地变化。因此，这些操作的需求被弱化了，以换取对其他更重要操作的性能优化：比如 get、put、containsKey 和 remove 等。

<!--这就是典型的根据场景，决定需求，舍弃相对不重要的东西，这种思想对于一个工科的工程师来说，都是非常重要的。-->

**在ConcurrentHashMap 中没有实现对Map 加锁以提供独占访问**。在Hashtable 和synchronizedMap中， 获得Map 的锁能防止其他线程访问这个Map。在一些不常见的情况中需要这种功能，例如通过原子方式添加一些映射， 或者对Map 迭代若干次并在此期间保持元素顺序相同。然而， 总体来说这种权衡还是合理的， **因为并发容器的内容会持续变化**。<!--【这是并发容器要面对的场景的核心特性】-->



### 额外的原子Map操作

**ConcurrentHashMap不能被加锁 来执行独占访问，因此我们无法使用客户端加锁来创建新的原子操作。**<!--惨兮兮，但是可以用第二种方式 -- 组合的方式对ConcurrentHashMap进行自定义拓展了-->但是一些常见的复合操作，例如"若没有则添加"，"若相等则移除"，"若相等则替换"等，都已经实现为原子操作并且在ConcurrentMap接口中声明。

### CopyOnWriteArrayList

CopyOnWriteArrayList 用于替代同步List, 在某些情况下它提供了更好的并发性能，**并且在迭代期间不需要对容器进行加锁或复制。**（类似地， CopyOnWriteArraySet 的作用是替代同步Set。)

“ 写入时复制(Copy-On-Write)" 容器的线程安全性在于， **只要正确地发布一个事实不可变的对象， 那么**

- 在访问该对象时，就不再需要进一步的同步。
- 在每次修改时， 都会创建并重新发布一个新的容器副本， 从而实现可变性。

“ 写入时复制 ” 容器的迭代器**保留一个指向底层基础数组的引用**， 这个数组当前位于迭代器的起始位置， 由于它不会被修改， 因此在对其进行同步时只需确保数组内容的可见性。因此， 多个线程可以同时对这个容器进行迭代， 而不会彼此干扰或者与修改容器的线程相互干扰。 “ 写入时复制 ” 容器返回的迭代器不会抛出ConcurrentModificationException, 并且返回的元素与迭代器创建时的元素完全一致， 而不必考虑之后修改操作所带来的影响。

显然， 每当修改容器时都会复制底层数组， 这需要一定的开销， 特别是当容器的规模较大时。 **仅当迭代操作远远多于修改操作时， 才应该使用 “ 写入时复制” 容器**。 这个准则很好地描述了许多事件通知系统：在分发通知时需要迭代已注册监听器链表， 并调用每一个监听器， 在 大多数情况下， 注册和注销事件监听器的操作远少于接收事件通知的操作。<!--给我的感觉是，像InnoDB事务中的MVCC和写锁，多个线程写的时候，不仅加上写锁，而且在MVCC中添加一个版本号-->

## 阻塞队列和生产者—消费者模式

阻塞队列提供了**可阻塞的put和take方法**，以及**支持实时的offer和poll方法**。

- 如果队列满了，那么put方法将阻塞直到有空间可用；
- 如果队列为空，那么take方法会阻塞到直到有元素可用。
- 队列可以是有界的也可以是无界的，无界队列永远都不会充满 ，因此put永远不会阻塞。

阻塞队列支持生产者—消费者这种设计模式，该模式把找出需要完成的工作与执行工作这两个过程分离开来，并把工作项放到一个"待完成"列表中以便随后处理，而不是找出后立即处理。它能简化开发过程，因为它消除了生产者类和消费者类之间的代码依赖性，此外，该模式还将生产数据的过程与使用数据的过程解耦开来以简化工作负载的管理，因为这两个过程在处理数据的速率上有所不同。

在基于阻塞队列构建的生产者—消费者设计中，当数据生成时，生产者把数据放入队列，而当消费者准备处理数据时，将从队列中获取数据。生产者不需要知道消费者的标识和数量，只需要将数据放队列即可，同样，消费者也不需要知道生着者是谁。BlockingQueue简化了生产者—消费者设计的实现过程，它支持任意数量的生产者和消费者。一种最常见的生产者—消费者设计模式就是线程池与工作队列的组合，在Executor任务执行框架中就体现了这种模式。这也是第6章和第8章的主题。

阻塞队列简化了消费者程序的编码，因为take操作会一直阻塞直到有可用的数据。如果生产者不能尽快地产生工作项使消费者保持忙碌，那么消费者就只能一直等待，直到有工作可做。<!--在某些情况下，例如服务器应用程序中，没有任何客户请求服务-->，而在其他一些情况下，这也表示需要调整生产者线程数量和消费者线程数量之间的比率，从而实现更高的资源利用率。（比如，在“网页爬虫”中，有无穷的工作需要完成）。

同样，put方法的阻塞特性也极大地简化了生产者的编码。如果使用有界队列，那么当队列充满时，生产者将阻塞并且不能继续生成工作，而消费者就有时间来赶上工作处理进度。

> 在构建高可靠的应用程序时，有界队列是一种强大的资源管理工具：它们能抵制并防止产生过多的工作项，使应用程序在负荷过载的情况下变得更加健壮。

虽然生产者-消费者模式能够将生产者和消费者的代码彼此解耦开来，但它们的行为仍然会通过共享工作队列间接地耦合在一起。

开发人员总会假设消费者处理工作的速率能赶上生产者生成工作项的速率，因此通常不会为工作队列的大小设置边界，但这将导致在之后需要重新设计系统架构。因此，应该尽早地通过阻塞队列在设计中构建资源管理机制—这件事情做得越早，就越容易。<!--在许多情况下，阻塞队列能使这项工作更加简单，如果阻塞队列并不完全符合设计需求，那么还可以通过信号量（Semaphore）来创建其他的阻塞数据结构（请参见5.5.3节）。在类库中包含了BlockingQueue的多种实现。-->

在类库中包含了BlockingQueue的多种实现

- LinkedBlockingQueue和ArrayBlockingQueue是FIFO队列，二者分别与LinkedList和ArrayList类似，但比同步List拥有更好的并发性能。
- PriorityBlockingQueue是一个按优先级排序的队列，当你希望按照某种顺序而不是FIFO来处理元素时，这个队列将非常有用。正如其他有序的容器一样，PriorityBlockingQueue既可以根据元素的自然顺序来比较元素（如果它们实现了Comparable方法），也可以使用Comparator来比较。
- SynchronousQueue实际上不是一个真正的队列，因为它不会为队列中元素维护存储空间。与其他队列不同的是，它维护一组线程，这些线程在等待着把元素加入或移出队列。如果以洗盘子的比喻为例，那么这就相当于没有盘架，而是将洗好的盘子直接放入下一个空闲的烘干机中。这种实现队列的方式看似很奇怪，但由于可以直接交付工作，**从而降低了将数据从生产者移动到消费者的延迟**。（在传统的队列中，在一个工作单元可以交付之前，必须通过串行方式首先完成入列[Enqueue]或者出列[Dequeue]等操作。）直接交付方式还会将更多关于任务状态的信息反馈给生产者。当交付被接受时，它就知道消费者已经得到了任务，而不是简单地把任务放入一个队列——这种区别就好比将文件直接交给同事，还是将文件放到她的邮箱中并希望她能尽快拿到文件<!--一个是及时的，一个是异步的-->。因为SynchronousQueue没有存储功能，因此put和take会一直阻塞，直到有另一个线程已经准备好参与到交付过程中。仅当有足够多的消费者，并且总是有一个消费者准备好获取交付的工作时，才适合使用同步队列。

### 示例：桌面搜索

有一种类型的程序适合被分解为 生产者 和 消费者，例如 代理程序，它将扫描本地驱动器上的文件并建立索引以便随后进行搜索，类似于某些桌面搜索程序或者 Windows 索引服务。

在 **程序清单 5-8 ** 的 `DiskCrawler` 中给出了一个生产者任务，即在某个文件层次结构中搜索 符合索引标准的文件，并将它们的文件名 放入 「工作队列」。而且 `Indexer` 中还给出了一个消费者任务，从队列中取出文件名称并对它们建立索引。

> **程序清单 5-8** 桌面搜索应用程序中的生产者任务和消费者任务

```java
// 一个关于 桌面搜索程序的对生产者和消费者的应用实例
public class ProducerConsumer {

    /**
     * 用来抓取桌面文件内容的生产者
     */
    static class FileCrawler implements Runnable {
        // 阻塞队列，用于存储要索引的文件
        private final BlockingQueue<File> fileQueue;
        // 文件的过滤器，判断是否要将文件放入队列
        private final FileFilter fileFilter;
        // 具体的被处理的队列中的生产资源，在这个例子中是file
        private final File root;

        public FileCrawler(BlockingQueue<File> fileQueue, FileFilter fileFilter, File root) {
            this.fileQueue = fileQueue;
            this.root = root;
            // 重写了 accept 方法
            this.fileFilter = f -> f.isDirectory() || fileFilter.accept(f);
        }

        // 返回是否已经对文件进行索引，但是这里没有进行判断，直接返回的就是 false
        private boolean alreadyIndexed(File file) {
            return false;
        }

        @Override
        public void run() {
            try {
                crawl(root);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        /**
         * 将桌面中的所有 文件放入 fileQueue 中
         *
         * @param root
         * @throws InterruptedException
         */
        private void crawl(File root) throws InterruptedException {
            File[] entries = root.listFiles(fileFilter);
            if (entries != null) {
                for (File entry : entries) {
                    if (entry.isDirectory()) {
                        crawl(entry);
                    }
                    // 但是这里 返回的肯定是 false 然后 变成 true 进入 put 逻辑
                    else if (!alreadyIndexed(entry)) {
                        fileQueue.put(entry);
                    }
                }
            }
        }
    }

    // 消费者，获取队列中的元素，这里是文件，然后对其进行索引（方法逻辑未实现，只是列出其逻辑）
    static class Indexer implements Runnable {
        private final BlockingQueue<File> queue;

        public Indexer(BlockingQueue<File> queue) {
            this.queue = queue;
        }


        @Override
        public void run() {
            try {
                while (true) {
                    indexFile(queue.take());
                }
            } catch (InterruptedException e) {
                // 发生异常就使线程停止
                Thread.currentThread().interrupt();
            }
        }

        public void indexFile(File file) {
            // 对文件进行索引编号的方法，这里书中并没有实现具体逻辑
        }
    }

   
}
```

**生产者 — 消费者 模式提供了一种 适合线程的方法将 桌面搜索问题分解为 更简单的组件**。 将 「文件遍历」 与建立索引等功能分解为独立的操作，比将所有功能都放到一个操作中实现有着更高的 「代码可读性」 和 「可重用」性：每个操作只需要完成一个任务，并且阻塞队列将负责所有的控制流，因此每个功能的代码都更加简单清晰。

<!--【也就是将逻辑控制完全交给了阻塞队列，从而使 生产 和 消费 2个动作独立，各自逻辑自己处理，分解成2个独立的块，更加清晰，这部分设计的意图我弄明白了，这是 生产者--消费者模式在代码设计上带来的好处】-->

生产者 — 消费者 模式同样能带来许多**「性能优势」**。 生产者 — 消费者 可以并发地执行。如果一个是 I/O 密集型，另一个是 CPU 密集型，那么并发执行的吞吐率要高于 串行执行的吞吐率。

如果生产者和消费者的「并行度」不同，那么将它们紧密耦合在一起会把整体并行度降低为二者中更小的并行度。

<!--【也就是木桶理论，性能最差的部分决定整个程序的性能】-->

在**程序清单 5-9** 中启动了多个爬虫程序和索引建立程序，每个程序都在各自的线程中运行。前面提到过，消费者线程永远不会退出，因而程序无法终止， [第7章](https://github.com/funnycoding/blog/issues/27) 将介绍多种技术来解决这个问题。<!--【并发中的终止问题】-->

虽然这个示例使用了 显示管理 的线程，**但许多 生产者—消费者 的设计也可以通过 `Executor` 任务执行框架来实现，其本身使用的也是 生产者 — 消费者模式。** **【 使用线程池 Executor 也间接的使用了 生产者 — 消费者 模式】**

> 程序清单 5-9 启动桌面搜索

```java
 // 设定队列的界限
    private static final int BOUND = 10;

    // 设置消费者的数量，这里是获取当前运行环境的CPU核心数并将其设置为消费者数量
    private static final int N_CONSUMERS = Runtime.getRuntime().availableProcessors();

    public static void startIndexing(File[] roots) {
        BlockingQueue<File> queue = new LinkedBlockingQueue<>(BOUND);

        FileFilter filter = pathname -> true;

        // 生产者,将文件放入 queue 中，填充要标记的文件队列
        for (File root : roots) {
            // 每个 roots 分配一个线程用来填充要进行标记的文件队列
            new Thread(new FileCrawler(queue, filter, root)).start();
        }

        // 消费者,消费队列中的文件，对其进行 indexFile() 标记方法
        for (int i = 0; i < N_CONSUMERS; i++) {
            new Thread(new Indexer(queue)).start();
        }
    }
```



### 串行线程封闭

在 `java.util.concurrent` Java 并发包中实现的各种阻塞队列 都包含了足够的 内部同步机制，从而安全地将对象从 「生产者线程」 发布到 「消费者线程」。

对于可变对象， 生产者—消费者 这种设计与阻塞队列一起，促进了**线程封闭**，从而将对象所有权从生产者交付给消费者。

线程封闭对象只能被单个线程持有，但可以通过安全地 「发布」 该对象来 "转移" 对象的所有权。 在转移所有权之后也只有另一个线程能获得这个对象的访问权限，并且发布对象的线程不会再访问它。

这种安全的发布确保了对象状态对于 新的所有者 来说是可见的，并且由于最初的所有者不会再访问它，因此对象被封闭在新的 线程中。

新的所有者线程可以任意对该对象进行修改，因为它具有 独占 的访问权。

「对象池」 利用了 串行线程封闭，将对象 "借给" 一个请求线程。 只要对象池内包含足够的 内部同步 来安全地发布池中的对象，并且只要客户代码本身不会发布池中对象，或者在将对象返回给对象池之后就不再使用它，那么就可以安全地在线程之间传递所有权。

<!-- 串行体现在流程上： 比如 生产者线程持有对象 -> 生产者线程放弃对象，交给Queue-> 消费者获取对象；又比如, 客户端持有对象 -> 客户端放弃对象，交给对象池 -> 对象池中某个线程获取对象。 看到没，这些流程都是串行进行的  -->

我们也可以使用其他发布机制来传递 可变对象 的所有权，**但必须确保只有一个线程能接受被转移的对象。**【基本条件】 阻塞队列简化了这项工作，除此之外，还可以通过 `ConcurrentMap` 的原子方法 `remove` 或者 `AtomicReference` 的原子方法 `compareAndSet` 来完成这项工作。

### 双端队列与工作密取

Java 6 增加了两种容器类型 ，`Deque`（发音为 "deck"）和 `BlockingDeque` ，它们分别对 `Queue` 和 `BlockingQueue` 进行了扩展。

Deque 是一个 「双端队列」，实现了在队列头和队列尾的**高效插入和移除**<!--也就是高效修改操作-->，具体实现包括 `ArrayDeque` 和 `LinkedBlockingDeque`。

正如阻塞队列适用于 "生产者—消费者" 模式，所有消费者有一个共享的工作队列，而在 「工作密取」设计中，每个消费者都有各自的双端队列。<!--从共享变为了独享-->

如果一个消费者完成了自己双端队列中的全部工作，那么它可以从其他消费者双端队列末尾 「秘密地获取工作」。<!--说明消费者可以获取其他消费者队列的末尾的数据-->

这种工作模式有优势的原因：

- 在 「大多数时候」，消费者只访问自己的双端队列，从而极大地减少了竞争。
-  当消费者线程需要访问另一个队列时，它会从队列的「尾部」 而不是 「头部」获取工作，因此进一步降低了队列上的竞争程度。

「工作密取」非常适用于既是消费者，也是生产者的问题 —— 当执行某个工作时可能导致出现更多的工作。

例如： 在网页爬虫程序中处理一个页面时，通常会发现有更多的页面需要处理。类似的还有很多 图 搜索的算法，例如在 垃圾回收阶段对 堆进行标记，都可以通过 工作密取 机制来实现 「高效并行」。 当一个工作线程找到新的任务单元时，它会将其放到自己队列的末尾（或者在**工作共享设计模式**中，放入其他工作者线程队列中）。

当「双端队列」 为空时，它会在另一个线程的队列队伍查找新的任务，从而确保每个线程都保持忙碌状态。**【这个设计可以很好的提高线程的利用率】**



## 阻塞方法与中断方法

线程可能会阻塞或暂停执行，原因有多种：

- **等待 I/O 操作结束**
- **等待获取一个锁**
- **等待从 Thread.sleep 方法中醒来**
- **等待另一个线程的计算结果**

当**线程阻塞**时，它通常被**「挂起」**，并处于某种**『阻塞状态』**（BLOCK、WAITING 或 TIMED_WAITING）。阻塞操作与执行时间很长的普通操作的差别在于 ： 被阻塞的线程必须等待某个不受它控制 的事件发生后 才能继续执行，例如「等待 I/O 操作完成」，「等待某个锁可用」，或者「等待外部计算的结束」。

当某个外部事件发生时，线程被置回 **RUNABLE** 可运行状态，并可以再次被调度执行。

`BlockingQueue` 的 `put` 和 `take` 等方法会抛出 **受检查异常**（**Checked Exception**） `InterruptedException`，这与类库中其他一些方法的做法相同，例如 `Thread.sleep`。

当某方法抛出 `InterruptedException` 时，表示该方法是一个**阻塞方法**，如果这个方法被中断，那么它将努力提前结束 「阻塞状态」。<---【阻塞方法的定义，比如 `Thread.sleep`】

`Thread` 提供了 `interrupt` 方法，用于中断线程或者查询线程是否已经被中断。 每个线程都有一个布尔类型的属性，表示线程的中断状态，当中断线程时，将设置这个状态。

中断是一种**「协作机制」**，一个线程不能强制其他线程停止正在执行的操作而去执行其他的操作。<---【也就是只能建议，而不能强制停止】

当 「线程A」 中断 「线程B」 时，A 仅仅是要求 B 在执行到某个可以暂停的地方停止正在执行的操作 —— 前提是 「线程B」 **愿意**停止下。虽然在 API 或者 语言规范中并没有为中断定义任何特定应用级别的语义，但最常使用中断的情况就是 「**取消某个操作**」。

方法对中断请求的响应度越高，就越容易及时取消那些执行时间很长的操作。<---【那么方法的响应度是可以设置的吗？ 总是把话说一半。。。】

当在代码中调用了一个将抛出 InterruptedException 异常的方法时，你自己的方法也就变成了一个阻塞方法，并且必须要处理对中断的响应，对于库代码来说，有两种基本选择：

- 传递`InterruptedException`。避开这个异常通常是最明智的策略 —— 只需要把 InterruptedException 抛出给方法的调用者。

  - 根本不捕获该异常。
- 捕获该异常，在执行某种简单的清理工作后再次抛出该异常。
  
- 恢复中断。 当不能抛出 `InterruptedException` 时，例如 当代码是 `Runnable` 的一部分时，这种情况下必须捕获 `InterruptedException`，并通过调用当前线程上的 `interrupt` 方法恢复中断状态，这样在调用栈中的更高层代码将看到引发了一个中断： 如**程序清单 5-10**

> 程序清单 5-10 恢复中断状态以屏蔽中断：

```java
// Restoring the interrupted status so as not to swallow the interrupt
// 恢复中断状态以避免中断被屏蔽
public class TaskRunnable implements Runnable {
    BlockingQueue<Task> queue;

    @Override
    public void run() {
        try {
            processTask(queue.take());
        } catch (InterruptedException e) {
            // 恢复被中断的状态 将调用者线程的中断状态设为true。
            Thread.currentThread().interrupt();
        }
    }
    // 对Task进行处理的业务逻辑代码
    void processTask(Task task) {
        // 处理 task
    }

    interface Task {
    }
}
```

还可以采用一些更复杂的终端处理方法，但是上述两种方法已经可以应付大多数情况了。 然而在出现 `InterruptedException` 时**不应该**做的事情是：**捕获它但是不做出任何响应**。【也就是异常的侵吞，这种操作在任何时候应该都是不推荐的】。这将使调用栈更上层的代码无法对中断采取处理措施，因为线程被中断的证据已经丢失。

只有在一种特殊的情况下才能屏蔽中断 —— 对 Thread 进行扩展，并且能够调用栈上更高层的代码。 [第7章](https://github.com/funnycoding/blog/issues/28) 将进一步介绍 **「取消」 和 「中断」** 等操作。

## 同步工具类

在容器类中，「**阻塞队列」是一种独特的类**：它们不仅作为保存对象的**容器**，还能协调生产者和消费者等线程之间的**控制流**。【也就是除了容器功能还带有逻辑控制功能】，因为 take 和 put 方法将阻塞，直到队列到达期望的状态。（队列既非空，也非满）。

同步工具类可以是任何一个对象，只要它根据自身的状态来协调线程的控制流。<---【这里任何一个对象 怎么理解？不是就那几个容器类吗？】

阻塞队列可以作为同步工具类，其他类型的同步工具类还包括：

- **信号量**（Semaphore）
- **栅栏**（Barrier）
- **闭锁**（Latch)。

在 JDK 中还包含其他一些同步工具类的类，如果这些类无法满足需要，那么可以按照 [第14章](https://github.com/funnycoding/blog/issues/29) 中给出的机制来创建自己的同步工具类。

所有的同步工具类都包含一些特定的「结构化属性」：它们封装了一些状态，这些状态将决定执行同步工具类的线程是继续执行还是等待，此外还提供了一些方法对「状态」进行操作，以及另一些方法用于高效地等待同步工具类进入到 「预期状态。」

###  闭锁

「闭锁」是一种「同步工具类」，可以延迟线程的进度直到其到达终止状态。

闭锁的作用相当于一扇门：在闭锁到达结束状态之前，这扇门一直是关闭的，并且没有任何线程能够通过，当到达结束状态时，这扇门会打开并允许所有线程通过。

当闭锁到达结束状态后，将不会再改变状态，因此这扇门将永远保持打开状态。<!--所以说这是一次性的-->

闭锁可以用来确保某些活动直到其他活动都完成才继续执行 ，例如：

> - 确保某个计算在其需要的所有资源都初始化之后才继续执行。 二元闭锁（包括两个状态） 可以用来表示 "资源 R 已经被初始化"，而所有需要 R 的操作都必须先在这个闭锁上等待。
> - 确保某个服务在其依赖的所有其他服务都启动之后才会启动。每个服务都有一个相关的 「二元闭锁」。 当启动服务 `S` 时，将首先在 `S` 依赖的其他服务的闭锁上等待，在所有依赖的服务都启动后会 释放闭锁`S`，这样其他依赖 `S` 的服务才能继续执行。
> - 等待直到某个操作的所有参与者（例如，在多玩家游戏中的所有玩家）都就绪再执行操作。在这种情况中，当所有玩家准备就绪时，闭锁达到结束状态。【也有种情况是，在等待时间到达预定时间后，比如吃鸡准备1分钟，则闭锁到达结束状态，游戏开始。

`CountDownLatch` 是一种灵活的 「闭锁实现」，可以在上述情况中使用，它可以使一个或多个线程等待一组事件发生。

闭锁状态包括一个「计数器」，该计数器被初始化为一个正数，表示需要等待的事件数量。 `countDown` 方法递减计数器，表示有一个事件已经发生了，而 `await` 方法等待计数器到达零，这表示所有需要等待的事件都已经发生。

如果计数器的值非零，**那么 `await` 会一直阻塞直到计数器为零**，或者**等待中的线程中断**，或者**等待超时**。

在程序清单 5-11 的 `TestHarness` 中给出了 闭锁的**两种常见用法** 。 `TestHarness` 创建一定数量的线程，利用它们并发地执行指定的任务。它使用两个闭锁，分别表示 "**起始门（Starting Gate）"** 和 **"结束门(Ending Gate)"**。

**起始门计数器的初始值为1**，而**结束门计数器的初始值为 「工作线程」 的数量**。 每个工作线程首先要做的事情就是在起始门上等待，从而确保所有线程都就绪后才开始执行。

而每个线程要做的最后一件事情就是调用 结束门 的 `countDown` 方法使计数器 -1 ，这能使主线程高效地等待直到所有工作线程都执行完成，因此可以统计所消耗的时间。

> 程序清单 5-11 在计时测试中使用 CountDownLatch 来启动和停止线程：

<!--与CountDownLatch的第一次交互是主线程等待其他线程。主线程必须在启动其他线程后立即调用CountDownLatch.await()方法。这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。CountDownLatch提供了await()方法来使当前线程一直等待，直到计数器的值减为0，或者线程被中断-->

```java
// Using CountDownLatch for starting and stopping threads in timing tests
public class TestHarness {
    public long timeTask(int nThreads, final Runnable task) throws InterruptedException{
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);

        for (int i = 0; i < nThreads; i++) {
            Thread t = new Thread() {
                @Override
                public void run() {
                    try {
                        startGate.await();//重点！！！
                        try {
                            task.run();//执行指定的任务task
                        } finally {
                            endGate.countDown();//重点！！！
                        }
                    } catch (InterruptedException e) {
                      
                    }
                }
            };
            // 使线程开始执行
            t.start();
        }
        long start = System.nanoTime();
        startGate.countDown();//重点！！！
        endGate.await();//重点！！！
        long end = System.nanoTime();
        return end - start;
    }
}
```

为什么要在 `TestHarness` 中使用闭锁而不是在线程创建后就立即启动？ 或许我们希望测试 N 个线程并发执行某个任务需要的时间。

如果在创建线程后立即启动它们，那么先启动的线程将 "领先" 后启动的线程，并且活跃线程数量会随着时间的推移而增加或减少，竞争程度也在不断发生变化。

「启动门 」 `startGate` 将使得 主线程能够同时释放所有工作线程，而 「结束门」 `endGate` 则使主线程能够等待最后一个线程执行完成，而不是顺序地等待每个线程执行完成。

### FutureTask

`FutureTask` 也可以用做闭锁（`FutureTask` 实现了 **Future**语义，**表示一种抽象的可生成结果的计算** )。

`FutureTask` 表示的计算是通过 `Callable` 来实现的，相当于一种可生成结果的 `Runnable`，并且可以处于以下 3种状态：

- **等待运行（Wating to run）**
- **正在运行(Running)**
- **运行完成(Completed)**

"执行完成" 表示计算的所有可能结束方式，包括正常结束、由于取消而结束和由于异常而结束等。

**当 FutureTask 进入完成状态后，它会永远停止在这个状态上。**

`Future.get` 的行为取决于任务的状态。如果任务已经完成，那么 get 会立即返回结果，否则 get 将阻塞直到任务进入完成状态，然后返回结果或者抛出异常。

`FutureTask` 将计算结果从「执行计算的线程」传递到「获取这个结果的线程」，而 `FutureTask` 的规范确保了这种传递过程能实现结果的安全发布。 **<--- 【那么 FutureTask 的规范是什么？】**

`FutureTask` 在 `Executor` 框架中表示**异步任务**，此外还可以用来表示一些时间较长的计算，这些计算可以在使用计算结果之前启动。

**程序清单 5-12** 中的 Preloader 就使用了 FutureTask 来执行一个高开销的计算，并且计算结果将在稍后使用。

通过**提前启动计算**，可以减少等待结果时需要的时间：

> **程序清单 5-12 使用 FutureTask 来提前加载稍后需要的数据**

```java
public class PreLoader {
    ProductInfo loadProductInfo() throws DataLoadException {
        return null;
    }

    // 使用 Future 提前调用 长耗时的方法 loadProductInfo()
    private final FutureTask<ProductInfo> future = 
      new FutureTask<ProductInfo>(new Callable<ProductInfo>() {
        @Override
        public ProductInfo call() throws Exception {
            return loadProductInfo();
        }
    });

    // FutureTask 继承于 Runnable
    private final Thread thread = new Thread(future);

    public void start() {thread.start();}

    public ProductInfo get() throws InterruptedException, DataLoadException {
        try {
            return future.get();
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            if (cause instanceof DataLoadException) {
                throw (DataLoadException) cause;
            } else {
                throw LaunderThrowable.launderThrowable(cause);
            }
        }
    }

    interface ProductInfo {}
}

class DataLoadException extends Exception {
}
```

`PreLoader` 创建了一个 `FutureTask`，其中包含从数据库加载产品信息的任务，以及一个执行运算的线程。

由于在构造函数或静态初始化方法中启动线程并不是一个好方法，因此提供了一个 `start` 方法来启动线程。

当程序随后需要 `ProductInfo` 时，可以调用 `get` 方法，如果数据已经加载，那么将返回这些数据，否则将等待加载完成后再返回。

`Callable` 表示的任务可以抛出 受检查的或未受检查 的异常，并且任何代码都可能抛出一个 `Error`。无论任务代码抛出什么异常，都会被封装到一个 `ExecutionException` 中，并在 `Future.get` 中被**重新抛出**。

这将使调用 `get` 的代码变得复杂，因为它不仅需要处理可能出现的 `ExecutionException`（以及未检查的 `CancellationException`） 而且还由于 `ExecutionException` 是作为一个 `Throwable` 类返回的，因此处理起来并不容易。 **【怎么理解？】**

在 `Preloader` 中，当 `get` 方法抛出 `ExecutionException`时，可能是以下三种情况之一：

- **Callable 抛出的受检查异常**
- **RuntimeException 运行时异常**
- **以及 Error**

我们必须对每种情况进行单独处理，但我们将使用 **程序清单 5-13** 中的 `launderThrowable` 辅助方法来封装一些 复杂的异常处理逻辑。

在调用 `launderThrowable` 之前， `Preloader` 会首先检查已知的受检查异常，并重新抛出它们。剩下的是未检查异常，`Preloader` 将调用 `launderThrowable` 并抛出结果。

如果 `Throwable` 传递给 `launderThrowable` 的是一个 `Error`，那么 `launderThrowable` 将直接再次抛出它；如果不是，`RuntimeException`，那么将抛出一个 `IllegalStateException` 表示这是一个逻辑错误。 剩下的 `RuntimeException`， `launderThrowable` 将把它们返回给调用者，而调用者通常会重新抛出它们。

**【这一大段都是对这个异常处理逻辑的描述，其实就是分情况就行了3个if判断然后进行处理】**

> **程序清单 5-13 强制将未检查的Throwable 转为 RuntimeException**

```java
public class LaunderThrowable {
    /**
     * Coerce an unchecked Throwable to a RuntimeException
     * <p/>
     * If the Throwable is an Error, throw it; if it is a
     * RuntimeException return it, otherwise throw IllegalStateException
     */
    public static RuntimeException launderThrowable(Throwable t) {
        if (t instanceof RuntimeException)
            return (RuntimeException) t;
        else if (t instanceof Error)
            throw (Error) t;
        else
            throw new IllegalStateException("Not unchecked", t);
    }
}
```

###  信号量

**计数信号量**（**Counting Semaphore**） 用来控制同时访问某个特定资源的操作数量，或者同时执行某个指定操作的数量。**计数信号量还可以用来实现某种资源池，或者对容器施加边界。**

`Semaphore` 中管理一组虚拟的许可（`permit`），许可的初始数量可以通过构造函数来指定。在执行操作时可以首先获得许可（只要还有剩余的许可），并在使用以后释放许可。

- 如果没有许可`permit`，那么 **`acquire` 将阻塞直到有许可**（或者直到被中断或者操作超时）。

- **`release` 将返回一个许可给信号量。**在这种实现中不包含真正的许可对象，并且 `Semaphore` 也不会将许可与线程关联起来，因此在一个线程中获得的许可可以在另一个线程中释放。 可以将 `acquire` 操作视为消费一个许可，而 `release` 操作是创建一个许可，`Semaphore` 并不受限于它在创建时的初始许可数量。

计算信号量的一种简化形式是 「二值信号量」，即**初始值为 1 的`Semaphore`**。 二值信号量可以用作互斥体（**mutex**），并具备**不可重入锁**<!--不要大意啊，和Synchronized不一样，是不可重入的-->的语义：谁拥有这个**唯一的许可**，谁就拥有了**「互斥锁」**。

> `Semaphore` **可以用于实现资源池，例如数据库连接池。**

我们可以构造一个**固定长度的资源池**，当池为空时，请求资源将会失败，但你真正希望看到的行为是阻塞而不是失败，并且当池变为非空状态时解除阻塞。

如果将 `Semaphore` 的计数值初始化为池的大小，并且从池中获取一个资源之前先调用 `acquire` 方法获取一个许可，在将资源返回给池之后调用 `release` 释放许可，那么 `acquire` 将一直阻塞直到资源池不为空。

- [第12章](https://github.com/funnycoding/blog/issues/29) 有界缓冲类中将使用`Semaphore` （在构造阻塞对象池时，一种更简单的方法是使用 `BlockingQueue` 来保存池的资源）

- 也可以使用 `Semaphore` 将任何一种容器变成 「有界阻塞容器」， 如 **程序清单 5-14** 中的 `BoundedHashSet` 所示，信号量的计数值会初始化为容器容量的最大值。 `add` 操作在向底层容器中添加一个元素之前，首先要获取一个许可。如果 `add` 操作没有添加任何元素，那么会立刻释放许可。同样 `remove` 操作释放一个许可，使更多的元素能够添加到容器中。底层的 `Set` 实现并不知道关于边界的任何信息，这是由 `BoundedHashSet` 来处理的。

> **程序清单 5-14 使用 Semaphore 为容器设置边界：**

```java
// 使用信号量 Semaphore 给 容器设置边界
public class BoundedHashSet<T> {
    private final Set<T> set;
    private final Semaphore sem;

    public BoundedHashSet(int bound) {
        this.set = Collections.synchronizedSet(new HashSet<>());
        // 设置边界
        sem = new Semaphore(bound);
    }

    public boolean add(T o) throws InterruptedException {
        sem.acquire();// 信号量 +1
        boolean wasAdded = false;
        try {
            wasAdded = set.add(o);
            return wasAdded;
        } finally {
            if (!wasAdded) {
                sem.release(); // 如果没有成功保存元素，则信号量 -1
            }
        }
    }

    public boolean remove(Object o) {
        boolean wasRemoved = set.remove(o);
        if (wasRemoved) {
            sem.release(); // 信号量 -1
        }
        return wasRemoved;
    }

    public static void main(String[] args) throws InterruptedException {
        BoundedHashSet<String> strSet = new BoundedHashSet<>(3);
        strSet.add("1");
        strSet.add("2");
        strSet.add("3");
        System.out.println(strSet.set);
        System.out.println(strSet.sem.toString());
        strSet.add("4");
    }
}
/**
输出
[1, 2, 3]
java.util.concurrent.Semaphore@6f94fa3e[Permits = 0]
*/
```

### 栅栏

- 闭锁是**一次性对象**，一旦进入终止状态，就不能被重置。

- **栅栏（Barrier) 类似于闭锁，它能阻塞一组线程直到某个事件发生。**

- 栅栏与闭锁的**关键区别**在于：所有线程都必须「同时」到达栅栏位置，才能继续执行。 **「闭锁用于等待事件，而栅栏用于等待其他线程。」**

栅栏用于实现一些协议，例如：几个家庭决定在某个地方集合"所有人决定6:00 在 麦当劳碰头到了以后要等其他人，等所有人到齐之后再讨论下一步要做的事情"。

`CyclicBarrier` 可以使一定数量的参与方 反复地在栅栏位置汇集，它在**并行迭代算法**中非常有用：这种算法通常将一个问题拆分成一系列相互独立的 子问题。当有线程都到达了栅栏位置时将调用 await 方法，这个方法将阻塞直到 所有线程都到达栅栏位置。

- 如果所有线程都到达了栅栏位置，那么栅栏将打开，此时所有线程都被释放，而栅栏将被**重置**以便下次使用。

- 如果对 await 的调用超时，那么 await 阻塞的线程被中断，那么栅栏就认为被打破，所有阻塞的 await 调用都将终止并抛出 `BrokenBarrierException`。

- 如果成功地通过栅栏，那么 await 将为每个线程返回一个唯一的到达索引号，我们可以利用这些索引来 "选举" 产生一个领导线程，并在下一次迭代中由该领导线程执行一些特殊的工作。

CyclicBarrier 还可以让你将一个栅栏操作传递给构造函数，这是一个 Runnable ，

- 当成功通过栅栏时会在一个子任务线程中执行它，
- 但在阻塞线程被释放之前，该任务是不能执行的。

在模拟程序中通常需要使用栅栏，例如某个步骤中的计算可以 「并行执行」 但必须等到该步骤中的所有计算都执行完毕才能进入下一个步骤。

例如，在 n-body 粒子模拟系统中，每个步骤都根据其他粒子的位置和属性来计算各个粒子的新位置。通过在每两次更新之间等待栅栏，能够确保在 第 k 步中的所有更新操作都已经计算完毕，才进入 第 k+1 步。

在程序清单 5-15 的 `CelluarAutomata` 中给出了如何通过栅栏来计算细胞的自动化模拟，在对模拟过程并行化时，为每个元素（在这个示例中相当于一个细胞）分配一个独立的线程是不现实的，因为这将产生过多的线程，而在协调这些线程上导致的开销将降低计算性能。

**合理的做法是：「将问题分解成一定数量的 子问题，为每个子问题分配一个线程来进行求解，之后再将所有的结果合并起来。」**

`CellularAutomate` 将问题分解为 **Ncpu** 个子问题，其中 **Ncpu** 等于当前环境下的可用 **CPU** 数量，并将给**每个子问题分配一个线程。**

> 在这种不涉及 I/O 操作 或共享数据访问的计算问题中，当线程数量为 CPU数量 或者 CPU + 1 的数量时将获得最优的吞吐量。 更多的线程不会带来任何帮助，甚至在某种程度上降低性能，因为多个线程会在 CPU 和 内存等资源上发生竞争

在每个步骤中，工作线程都为各自子问题中的所有细胞计算新值。当所有工作线程都到达栅栏时，栅栏会把这些新值提交给数据模型。

在栅栏的操作执行完以后，工作线程将开始下一步的计算，包括调用 `isDone` 方法来判断是否需要进行下一次迭代。

> **程序清单 5-15** 通过 **CyclicBarrier** 协调细胞自动衍生系统中的计算：

```java
// 使用栅栏协调计算细胞衍生系统
public class CellularAutomate {
    private final Board mainBoard;
    private final CyclicBarrier barrier;
    private final Worker[] workers;

    public CellularAutomate(Board board) { //初始化
        this.mainBoard = board;
      
        int count = Runtime.getRuntime().availableProcessors();
      
        this.barrier = new CyclicBarrier(count, new Runnable() {
            @Override
            public void run() {
                mainBoard.commitNewValues();
            }
        });
        
        this.workers = new Worker[count];
        for (int i = 0; i < count; i++) 
        		workers[i] = new Worker(mainBoard.getSubBoard(count, i));
        
    }

    private class Worker implements Runnable {
        private final Board board;

        public Worker(Board board) {
            this.board = board;
        }

        @Override
        public void run() {
            while (!board.hasConverged()) {
                for (int x = 0; x < board.getMaxX(); x++) 
                    for (int y = 0; y < board.getMaxY(); y++) 
                        board.setNewValue(x, y, computeValue(x, y));
                try{
                    barrier.await();//计数器加1，等待其他线程执行到同样的地方
                } catch(...){
                  ...
                }
            }
        }
    }

    public void start() { //启动部分
        for (int i = 0; i < workers.length; i++) {
            new Thread(workers[i]).start();
        }
        mainBoard.waitForConvergence();
    }
}
```

另一种形式的栅栏是 Exchanger，它是一种两方（Two-Party）栅栏，各方在栅栏位置上交换数据。当两方执行不对称操作时， Exchanger 会非常有用。

例如当一个线程向缓冲区写入数据，而另一个线程从缓冲区读取数据。这些线程可以使用 Exchanger 来汇合，并将写满的缓冲区与空的缓冲区交换。当两个线程通过 Exchanger 交换对象时，这种交换就把这两个对象安全地发布给另一方。

数据交换的时机取决于应用程序的响应需求。最简单的方案是：当缓冲区被填满时，由填充任务进行交换，当缓冲区为空时，由清空任务进行交换。

**这样会把需要交换的次数降至最低。**

但如果新数据的到达率不可预测，那么一些数据的处理过程就将延迟。另一个方法是：不仅当缓冲区被填满时进行交换，并且当缓冲区被填充到一定程度并保持一定的时间后，也进行交换。

## 构建高效且可伸缩的结果缓存

几乎所有的服务器应用程序都会使用 「某种形式的缓存」。重用之前的计算结果能降低延迟，提高吞吐量，但却需要消耗更多的内存。【空间与时间之间的转换】

像许多"重复发明的轮子"一样，缓存看上去非常简单。然而，简单的缓存可能会将 「性能瓶颈」 转换成 「可伸缩瓶颈」，即使缓存是用于提升单线程的性能，本节我们将开发一个 「高效」 且 「可伸缩」 的缓存，用于改进一个**高计算开销**的函数。

我们首先从简单的 `HashMap` 开始，然后分析它的**「并发性缺陷」**，并讨论如何修复它们。

在程序清单 5-16 的 `Computable<A,V>` 接口中声明了一个 函数 `Computable`，其输入类型为A，输出类型为V。

在 `ExpensiveFunction` 中实现的 `Computable`，需要很长时间来计算结果，我们将创建一个 Computable 包装器，帮助记住之前的计算结果，并将缓存过程封装起来。（这项技术被称为 `Memorization`）

> **程序清单 5-16** 使用 `HashMap` 和**「同步机制」**来初始化缓存

```java
interface Computable<A, V> {
    V compute(A arg) throws InterruptedException;
}

class ExpensiveFunction implements Computable<String, BigInteger> {
    @Override
    public BigInteger compute(String arg) throws InterruptedException {
        // 经过了长时间的计算之后
        return new BigInteger(arg);
    }
}

public class Memoizer1<A, V> implements Computable<A, V> {
    @GuardedBy("this")
    private final Map<A, V> cache = new HashMap<>();

    private final Computable<A, V> c;

    public Memoizer1(Computable<A, V> c) {
        this.c = c;
    }

    @Override
    public synchronized V compute(A arg) throws InterruptedException {
        V result = cache.get(arg);
        if (result == null) {
          	result = c.compute(arg);
            cache.put(arg, result);
        }
        return result;
    }
}
```

HashMap 是非线程安全的，

- 因此要确保两个线程不会同时对 HashMap 进行访问 ，Memoizer1 采用了一种保守的方法： 「对整个 `compute` 进行同步」。 
- 这种方法能确保线程安全性，但会带来一种明显的 「可伸缩性」 问题：每次只有一个线程能执行 `compute`方法，并且其可能是一个长耗时的方法，那么其他调用 `compute` 的线程可能被阻塞很长时间。并且这种长时间的阻塞没法通过提升硬件数目来解决，因为就算核心或线程再多，也只有一个线程能访问该方法。

如果有多个线程在排队等待还未计算出的结果，那么 `compute` 方法的计算时间可能比没有 "记忆" 操作的计算时间更长。

<img src="https://chengfeng96.com/blog/2018/10/24/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%A4%BA%E4%BE%8B%E4%B9%8B%E6%9E%84%E5%BB%BA%E9%AB%98%E6%95%88%E4%B8%94%E5%8F%AF%E4%BC%B8%E7%BC%A9%E7%9A%84%E7%BB%93%E6%9E%9C%E7%BC%93%E5%AD%98/1.png" style="zoom:33%;" />

**程序清单 5-17** 中的 `Memoizer2` 用 `ConcurrentHashMap` 代替了 `HashMap` 来改进 `Memoizer1` 中糟糕的并发行为。由于 `ConcurrentHashMap` 是线程安全的，因此在访问底层 `Map` 时就不需要进行同步，因而避免了 `Memoizer1` 中的 `compute`方法上使用内置锁导致的串行性。

`Memoizer2` 比 `Memoizer1` 有更好的并发行为：多线程可以并发地使用它。但它在作为缓存时仍然有一些不足 —— 当**「两个线程同时调用 `compute` 时存在一个漏洞」**：**可能会导致计算得到相同的值。**

在使用 memoization 的情况下，这只会带来低效，因为缓存的作用是避免相同的数据被计算多次。但对于更通用的缓存机制来说，这种情况（在某种情况下导致计算得到的值相同的情况）将更为糟糕。

对于只提供单次初始化的缓存来说，这个漏洞就会带来安全风险。

> **程序清单 5-17** 用 `ConcurrentHashMap` 替换 `HashMap`：

```java
public class Memoizer2<A, V> implements Computable<A, V> {
    private final Map<A, V> cache = new ConcurrentHashMap<>();
    private final Computable<A, V> c;

    public Memoizer2(Computable<A, V> c) {
        this.c = c;
    }

    @Override
    public V compute(A arg) throws InterruptedException {
        V resul = cache.get(arg);
        if (resul == null) {
            resul = c.compute(arg);
            cache.put(arg, resul);
        }
        return resul;
    }
}
```

`Memoizer2` 的问题在于，如果某个线程启动了一个 「开销很大」 的计算，而其他线程并不知道这个计算正在进行，那么很可能会重复这个计算。

<img src="https://chengfeng96.com/blog/2018/10/24/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%A4%BA%E4%BE%8B%E4%B9%8B%E6%9E%84%E5%BB%BA%E9%AB%98%E6%95%88%E4%B8%94%E5%8F%AF%E4%BC%B8%E7%BC%A9%E7%9A%84%E7%BB%93%E6%9E%9C%E7%BC%93%E5%AD%98/2.png" style="zoom:50%;" />

我们希望通过某种方法来表达"线程X 正在计算 f(1)" 这种情况，这样当另一个线程查找 f(1) 时，它能够知道「最高效」 的方法是等待 「线程X」 计算结束，然后再去查询缓存获取 f(1) 的值。

**已经有一个类能够基本实现这个功能：`FutureTask`。**

`FutureTask` 表示一个 **「计算的过程」**，这个过程可能已经计算完成，也可能正在进行。 如果有结果可用，那么 `FutureTask.get` 将立即返回结果，否则它会一直阻塞，直到结果计算出来再次将其返回。

程序清单 5-18 中的 `Memoizer3` 将用于缓存值的 Map 重新定义为 `ConcurrentHashMap<A,Future<V>>` ，替换原来的 `ConcurrentHashMap<A,V>` 。

`Memoizer3` 首先检查某个相应的计算是否已经开始（ `Memoizer2` 与之相反，它首先判断某个计算是否已经完成）。

如果对应的值在缓存中不存在，并且对其进行的计算还没有启动，那么就创建一个 `FutureTask`，并注册到 `Map` 中，然后启动计算，如果针对该值的计算已经启动，那么等待现有计算的结果，结果可能会很快得到，也可能还在运算过程中，但这堆 `Future.get` 的调用者来说是「透明」的【也就是不需要对其进行显式的处理，就调用 `Future.get` 就完事了】。

> **程序清单 5-18** 基于 `FutureTask` 的 `Memoizing` 封装器：

```java
public class Memoizer3<A, V> implements Computable<A, V> {
    private final Map<A, Future<V>> cache = new ConcurrentHashMap<>();
    private final Computable<A, V> c;

    public Memoizer3(Computable<A, V> c) {
        this.c = c;
    }

    public V compute(A arg) throws InterruptedException {
        Future<V> f = cache.get(arg);

        if (f == null) {
            Callable<V> eval = () -> c.compute(arg);

            FutureTask<V> ft = new FutureTask<>(eval);
            f = ft;
            cache.put(arg, ft);
            ft.run(); //这里调用 c.compute 方法
        }
        // 获取 FutureTask 的计算的值，如果正在计算中，则阻塞，直到其值返回
        try {
            return f.get();
        } catch (ExecutionException e) {
            throw LaunderThrowable.launderThrowable(e.getCause());
        }
    }
}
```

`Memoizer3` 的实现几乎是完美的：它表现出非常好的并发性（源于 `ConcurrentHashMap` 高效的并发性）。

- 同时若结果已经被计算出来，那么将立即返回。

- 如果其他线程正在计算该结果，那么新到的线程将一直等待这个结果被计算出来。

- 它只有一个「缺陷」：仍然存在两个线程计算出相同值的漏洞。 这个漏洞的发生概率要远小于 `Memoizer2` 中发生的概率，但由于 `compute` 方法 中的 if 代码块仍然**是非原子的（nonatomic) 的"先检查，再执行的操作"**，因此两个线程仍然有可能在同一时间内调用 compute 来计算相同的值，即「二者都没有在缓存中找到期望的值，因此都开始计算」。

这个错误的执行时序如 **图5-4** 所示：

<img src="https://chengfeng96.com/blog/2018/10/24/Java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%A4%BA%E4%BE%8B%E4%B9%8B%E6%9E%84%E5%BB%BA%E9%AB%98%E6%95%88%E4%B8%94%E5%8F%AF%E4%BC%B8%E7%BC%A9%E7%9A%84%E7%BB%93%E6%9E%9C%E7%BC%93%E5%AD%98/3.png" style="zoom:50%;" />



`Memoizer3` 中存在这个问题的原因是，复合操作「若没有则添加」 是在底层的 `Map` 对象上执行的，而这个对象无法通过加锁来确保原子性。

程序清单 `5-19` 中的 `Memoizer` 使用了 `ConcurrentMap` 中的原子方法 `putIfAbsent`，避免了 `Memoizer3` 中两个线程计算相同值的漏洞：

> **程序清单 5-19** `MeMoizer` 的最终实现：

```java
public class Memoizer<A, V> implements Computable<A, V> {
    private final ConcurrentMap<A, Future<V>> cache = new ConcurrentHashMap<>();
    private final Computable<A, V> c;

    public Memoizer(Computable<A, V> c) {
        this.c = c;
    }

    @Override
    public V compute(A arg) throws InterruptedException {
        while (true) {
            Future<V> f = cache.get(arg);

            if (f == null) {
                Callable<V> eval = () -> c.compute(arg);
                FutureTask<V> ft = new FutureTask<>(eval);

                f = cache.putIfAbsent(arg, ft);
                if (f == null) {
                    f = ft;
                    ft.run(); // 执行 c.compute 具体计算逻辑
                }
            }
            try {
                return f.get();
            } catch (CancellationException e) {
                cache.remove(arg, f);
            } catch (ExecutionException e) {
                throw LaunderThrowable.launderThrowable(e.getCause());
            }
        }
    }
}
```

当缓存的是 `Future` 而不是**具体的值**时，将导致**缓存污染（Cache Pollution）**问题：如果某个计算被取消或者计算过程中失败，那么在计算这个结果时将指明计算过程被取消或者失败。

为了避免这种情况，如果 Memoizer 发现计算被取消，那么将把 Future 从缓存中移除。如果检测到 RuntimeException，那么也会移除 Future，这样将来的计算才可能成功。

`Memoizer` 同样没有解决

- **「缓存逾期」** 问题，但它可以通过 `FutureTask` 的子类来解决，**「在子类中为每个结果指定一个逾期时间，并定期扫描缓存中的逾期元素」**。
- 同样，它也没有解决「缓存清理」的问题，没有移除旧的计算结果以便为新的计算结果腾出空间，从而使缓存不会消耗过多的内存【这是这个缓存类的2个硬伤。】

在完成并发缓存的实现后，就可以为 [第2章](https://github.com/funnycoding/blog/issues/30) 中 「因式分解Servlet」 添加结果缓存。

程序清单 5-20 中的 `Factorizer` 使用 `Memoizer` 来缓存之前的计算结果，这种方式不仅**高效，而且可扩展性也更好：**

> 程序清单 5-20 在因式分解 servlet 中使用 Memoizer 来缓存结果：

```java
public class Factorizer extends GenericServlet implements Servlet {
    private final Computable<BigInteger, BigInteger[]> c = new Computable<BigInteger, BigInteger[]>{
    	public BigInteger[] compute(BigInteger arg){
    		return factor(arg);
    	}
    }

    private final Computable<BigInteger, BigInteger[]> cache = new Memoizer<>(c);

    @Override
    public void service(ServletRequest req, ServletResponse resp) {
        try {
            BigInteger i = extractFromRequest(req);
            // 从缓存中获取值，如果没有则计算值
            encodeIntoResponse(resp, cache.compute(i));
        } catch (InterruptedException e) {
            encodeError(resp, "factorization interrupted");
        }
    }
}
```

## 第一部分小结

目前为止，第一部分就结束了。这一部分中介绍了许多关于并发的基础知识，下面这个 "并发技巧清单" 列举了在第一部分中介绍的主要概念和规则：

- 可变状态是至关重要的（It's the mutable state,stupid)
  - 所有的并发问题都可以总结为如何协调对并发状态的访问。 可变状态越少，就越容易确保线程安全性。【当没有可变状态时，该类就是安全的，不存在线程安全性问题】
- 尽量将域声明为 final 类型的，除非确实需要它们是可变的。
- 不可变对象一定是线程安全的。
  - 不可变对象能极大地降低并发编程的复杂性。它们更加简单而且安全，可以任意共享而无须使用加锁或者保护性复制机制。
- 封装有助于管理复杂性。
  - 在编写线程安全的程序时，虽然可以将所有数据都保存在全局变量中，但是极不推荐这样做。将数据封装在类中，可以缩小对数据的访问路径，更易于维持不变性条件：将同步机制封装在对象中，更易于遵循同步策略。
- 用锁来保护每个可变变量。
- 当保护同一个不变性条件中的所有变量时，要使用 「同一个」 锁。
- 在执行 「复合操作」，要持有锁。
- 如果从多个线程中访问访问访问同一个 「可变变量」 时没有同步机制，那么程序会出现问题。
- 不要故作聪明地推断出不需要使用同步
- 在设计过程中考虑线程安全，或者在文档中明确指出这个类不是线程安全的。
- 将同步策略文档化。