---
title: chapter3对象的共享
date: 2020-11-13 09:19:03
tags: [并发]
---

ref: 

1. java如何理解隐式地使this引用逸出
   https://segmentfault.com/q/1010000007900854
   https://www.cnblogs.com/viscu/p/9790755.html

Abstract：第二章开头指出，要编写正确的并发程序，关键在与：在访问共享的可变状态时，需要正确的管理。第二章通过**同步**，避免多个线程同时访问相同的变量 (在《高性能MySQL》书中，这叫做**避免并发**)；而本章介绍如何**共享**和**发布对象**，从而使多个线程可以同时安全的访问相同的变量。这两章合在一起就形成了构建线程安全类以及通过java.util.concurrent类库来构建并发应用程序的重要基础。

[synchronized不仅仅只有原子性，还具有内存可见性]()。我们不仅希望防止某个线程正在使用对象状态而另一个线程在同时修改该状态，而且希望确保当一个线程修改了对象状态后，其他线程能够看到发生的状态变化<!--拓展：这种内存可见性像《高性能MySQL》书中所说的Read Uncomitted事务隔离级别的可见性-->。如果没有同步，那么这种情况就无法实现。你可以通过显式地同步或者类库中内置的同步来保证对象被安全地发布。

<!--这里注意：synchronized保障了变量不可修改，但是这个变量必须在同步代码块执行完了，才能被其他线程访问到最新变化。所以，我现在希望这个变量的变化，在同步代码块还没执行完时，就能及时被其他线程获取到最新数据，怎么办呢？就是volatile,使用可见性-->

## 可见性

```java
public class NoVisibility {
    private static boolean ready;
    private static int number;

    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready)
                Thread.yield();
//这里可能输出0，也可能永远都不会输出
            System.out.println(number);
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```

这里可能会产生两个问题

- 一是程序可能一直保持循环，因为对于读线程来说，ready的值可能永远不可见。
- 二是输入的number为0，这是因为重排序引起的，在写线程将ready与number从工作内存中写回到主内存中时，在没有同步的机制下，先写ready还是先写number这是不确定的，也就是说将它们写回到主内存时的顺序可能与程序逻辑顺序恰好相反，这是因为在单个线程下，只要重排序不会对结果产生影响，这是允许的。
  <!--在没有同步的情况下，编译器、处理器以及运行时等都可能对操作的执行顺序进行一些意想不到的调整。在缺乏足够同步的多线程程序中，要想对内存操作的执行顺序进行判断，几乎无法得出正确的结论。-->

### 失效数据

除非在每次访问变量时都使用同步，否则很可能获得该变量的一个失效值。更糟糕的是，失效值可能不会同时出现：一个线程可能获得某个变量的最新值，而获得另一个变量的失效值。

上面NoVisibility程序在多线程环境下还可能读取到过期数据，比如当ready为true时，写线程已将number域的值置为了42，但在它还未来得及将这个新值从工作内存中写回到主内存前，读线程就已将ready从主内存中读取出来了，这时的值还是为初始的默认值0，这个值显然是一个已过期了的值，因为number现在真的值应该为42，而不是0。

<!--拓展：在没有同步的情况下读取数据类似于数据库中使用READ_UNCOMMITTED（未提交读）隔离级别，这时你更愿意用准确性来交换性能。-->

在NoVisibility中，过期数据可能导致它打印错误数值，或者程序无法终止。过期数据可能会使对象引用中的数据更加复杂，比如链指针在链表中的实现。过期数据还可能引发严重且混乱的错误，比如意外的异常，脏的数据结构，错误的计算和无限的循环。

下面的程序更对过期数据尤为敏感：如果一个线程调用了set，但还未来得及将这个新值写回到主内存中时，而另一个线程此时正在调用get，它就可能看不到更新的数据了：

```java
@NotThreadSafe
public class MutableInteger {
    private int value;
    public int  get() { return value; }
    public void set(int value) { this.value = value; }
}
```

我们可以将set与get同步，使之成为线程安全的。注，仅仅同步某个方法是没有用的。

### 非原子的64位操作

非volatile类型的64位数值变量（double和long）。Java内存模型要求，变量的读取操作和写入操作都必须是原子操作，当对于非volatile类型的long和double变量，JVM允许将64位的读操作或写操作分解为两个32位的操作。当读取一个非volatile类型的long变量时，如果对该变量的读操作和写操作在不同的线程中执行，那么很可能会读取到某个值的高32位和另一个值得低32位。因此，即使不考虑失效数据问题，在多线程程序中使用共享且可变的long和double等类型的变量也是不安全的，除非用关键字volatile来声明它们，或者用锁保护起来。



### 锁和可见性

内置锁可以用来确保一个线程以某种可预见的方法看到另一个线程的影响，像下图一样。当B执行到与A相同的锁监视的同步块时，A在同步块之中所做的每件事，对B都是可见的，如果没有同步，就没有这样的保证。
<img src="https://img-blog.csdnimg.cn/20190316230729391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5OTUxOTgz,size_16,color_FFFFFF,t_70" style="zoom:67%;" />

现在我们可以进一步理解为什么在访问某个共享且可变的变量时要求所有线程在**同一个锁上同步**，就是为了确保某个线程写入该变量的值对于其他线程来说都是可见的。否则，如果一个线程在未持有正确锁的情况下去读某个变量，那么读到的可能是一个失效值。

> 加锁的含义不仅仅局限于互斥行为，还包括内存可见性。为了确保所有线程都能看到共享变量的最新值，所有执行读操作或者写操作的线程都必须在同一个锁上同步。

### volatile变量

volatile是一种弱同步的形式，它确保对一个变量的更新后对其他线程是可见的。当一个域声明为volatile类型后，编译器与运行时会监视这个变量：它是共享的，而且对它的操作不会与其他的内存操作一起被重排序。volatile变量不会缓存在寄存器或者缓存其他处理器隐藏的地方 ==> 所以，**读一个volatile类型的变量时，总会返回由某一线程所写入的最新值**。

**读取volatile变量的操作不会加锁，也就不会引起执行线程的阻塞**，<!--允许并发访问进行读-->这使得volatile变量相对于sychronized而言，只是轻量级的同步机制<!--针对可见性而言-->。

volatile变量对可见性的影响所产生的价值远远高于变量本身。线程A向volatile变量写入值，随后线程B读取该变量，所有A执行写操作前可见的变量的值，在B读取了这个volatile变量后，对B也是可见的（与解锁前所有动作对后继加锁后的动作可见是一样的）。所以[从内存可见性的角度来看]()，写入volatile变量就像退出同步块，读取volatile变量就像进入同步块。但是我们并不推荐过度依赖volatile变量所提供的可见性。因为依赖volatile变量来控制状态可见性的代码，比使用锁的代码更脆弱，更难以理解。

> 仅当volatile变量能简化代码的实现以及对同步策略的验证时，才应该使用它们。如果在验证正确性时需要对可见性进行复杂的判断，那么就不要使用volatile变量。volatile变量的正确使用方式包括：确保它们自身状态的可见性，确保它们所引用对象的状态的可见性，以及标示一些重要的程序生命周期事件的发生（例如，初始化或关闭）

```java
//一定要加上volatile，否则其他线程更新后可能不可见，因为必须等到执行线程同的步代码块全部执行完，其他线程才能看见
volatile boolean asleep;
while (!asleep)
    countSomeSheep();
```

volatile变量固然方便，但也存在限制，它们通常被当作标识完成、中断、状态的标记使用，比如上面程序中的asleep变量。尽管volatile也可以用来标示其他类型的状态信息，但是**决定这样做之前请格外小心，如volatile的语义不足以使用自增操作（i++）原子化。**

> 加锁可以保证可见性与原子性；volatile变量只能保证可见性。

只有满足了下面所有的标准后，你才能使用volatile变量：<!--其实就是不需要保证原子性，只需要可见性-->
1、 写入变量时并不依赖变量的当前值；或者能够能够确保只有单一线程修改变量的值；
2、 变量不需要与其他的状态变量共同参与不变约束；
3、 访问变量时，没有其他的原因需要加锁。



## 发布与逸出

**发布**一个对象的意思是，使对象能够被当前作用域之外的代码中使用。比如将一个指向该对象的引用存储到其他代码可以访问的地方、在一个非私有的方法中返回这个引用、也可以把它传递到共他类的方法中。在很多情况下，我们需要确保对象及它们的内部状态不被暴露，在另外一些情况下，为了正当的使用目的，我们又的确希望发布一个对象，这时为了线程安全可能需要同步。如果变量发布了内部状态，就可能危及到封装性，并使用程序难以维持稳定；如果发布对象时，它还没有完成构造，同样危及线程安全。一个对象在尚未准备地时就将它发布，这种情况称作**逸出**。下面看看一个对象是如何逸出的。

### 直接发布对象

最常见的发布对象的方式是将对象的引用存储到公共静态域，任何类和线程都能看到这个域。initialize方法实例化一个新的HashSet实例，并通过将它存储到knownSecrets引用，从而发布了这个实例：

```java
//发布对象
public static Set<Secret> knownSecrets;
public void initialize() {
    knownSecrets = new HashSet<Secret>();
}
```

### 间接发布对象

发布一个对象还会间接地发布其他对象。如果你将一个Secret对象加入集合knownSecrets中，你就已经发布了这个对象，因为任何代码都可以遍历并获得新Secret对象的引用。类似地，从非私有方法中返回引用，也能发布返回的对象，下面发布了包含洲名的数组，而这个数组本应是私有的：

```java
//内部可变的数据逸出（不要这样做）
class UnsafeStates {
    private String[] states = new String[] {
        "AK", "AL" ...
    };
    public String[] getStates() { return states; }
}
```

以这种方式发布states会出问题，这样会允许内部可变的数据逸出，请不要这样做。因为任何一个调用者都能修改它的内容。在这个例子中，数组states已经逸出了它所属的范围，这个本就是私有的数据，事实上已经变成公有的了。

**当发布一个对象时，在该对象的非私有域中引用的所有对象同样会被发布。一般来说，如果一个已经发布的对象通过非私有的变量引用和方法调用到达其他的对象，那么这些对象也都会被发布。**

假定有一个类C，对于C来说，“外部方法”是指行为并不完全由C来规定的方法，包括其他类中定义的方法以及类C中可以被改写的方法（既不是私有[private]方法也不是终结[final]方法）。当把一个对象传递给某个外部方法时，就相当于发布了这个对象。

无论其他的线程会对已发布的线程执行何种操作，其实都不重要，因为误用该引用的风险始终存在。当某个对象逸出后，你必须假设有某个类或线程可能会误用该对象。

最后一种发布对象和它的内部状态的机制是发布一个内部类实例。

```java
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener(
            new EventListener() {//会过早地暴露this
                public void onEvent(Event e) {
                    doSomething(e);
                }
            });
    }
}
```

当ThisEscape发布EventListener时，也隐含地发布了ThisEscape实例本身，因为在这个内部类的实例中包含了对ThisEscape实例的隐含引用。

### 安全对象构造过程

[当且仅当对象的构造函数返回时，对象才处于可预测的和一致的状态]()。从构造函数内部发布的对象，只是一个未完成构造的对象。甚至即使是在构造函数的最后一行发布的引用也是如此。如果this引用在构造器中逸出，这样的对象被认为是“没有正确构建的”，所以不要让this引用在构造期间逸出。

> 不要在构造构造过程中使this引用逸出

一个导致this引用在构造期间逸出的常见错误，是在构造函数中创建局部、匿名线程并启动它或者启动一个线程并显示地将this传递过去，这都是不安全的，因为新的线程在所属对象完成构造前就能看见了。在构造器中创建线程并没有错，但是最好不要立即启动它，取而代之的是，发布一个start或initialize方法来启动对象拥有的线程。

另外，构造器中调用一个覆盖的实例方法（既不是私有方法，也不是终结方法）同样会导致this引用在构造期间逸出。

如果想要在构造器中注册监听器或启动线程，你可以使用一个私有的构造函数和一个公有的工厂方法，这样避免了不正确的构造过程。
下面是使用工厂方法防止this引用在构造期间逸出：

```java
public class SafeListener {
    private final EventListener listener;

    private SafeListener() {//私有构造器
        listener = new EventListener() {
            public void onEvent(Event e) {
                doSomething(e);
            }
        };
    }

    //使用静态 工厂方法安全发布对象
    public static SafeListener newInstance(EventSource source) {
        SafeListener safe = new SafeListener();//等构造完后再注册
        source.registerListener(safe.listener);
        return safe;//安全发布对象
    }
}
```

> 只有当构造函数返回时，this引用才应该从线程中逸出。构造函数可以将this引用保存到某个地方，只要其他线程不会在构造函数完成之前使用它。



## 线程封闭

当访问共享的可变数据时，通常需要使用同步。一种避免使用同步的方式就是不共享数据。如果仅在单线程内访问数据，就不需要同步。这种技术被称为线程封闭。它是实现线程安全性最简单的方式之一。当某个对象封闭在一个线程中时，这种用法将自动实现线程安全性，即使被封闭的对象本身不是线程安全的。

在Java语言中并没有强制规定某个变量由锁来保护，同样在Java语言中也无法强制将对象封闭在某个线程中。线程封闭是在程序设计中的一个考虑因素，必须在程序中实现。Java语言及其核心库提供了一些核心机制来帮助维持线程封闭性，例如局部变量和ThreadLocal类，即便如此，程序员仍然需要负责确保封闭在线程中的对象不会从线程中逸出。

### Ad-hoc 线程封闭

Ad-hoc 线程封闭是指，维护线程封闭性的职责完全由程序实现来承担。由于Ad-hoc 线程封闭技术的脆弱性，因此在程序中尽量很少用它，在可能的情况下，应该使用更强的线程封闭技术（例如，栈封闭或ThreadLocal类）。

### 栈封闭

在栈封闭中，只能通过局部变量才能访问对象。局部变量的固有属性之一就是封闭在执行线程中。它们位于执行线程的栈中<!--更准确的说，是栈帧中-->，其他线程无法访问这个栈。

局部变量是线程安全的，只要我们不要将它们逸出。

### ThreadLocal 类

这个类能使先回城中的某个值与保存值的对象关联起来。ThreadLocal提供了get与set等访问接口或方法，这些方法为每个使用该变量的线程都存有一份独立的副本，因此get总是返回由当前线程在调用set时设置的最新值。

ThreadLocal对象通常用于防止对可变的单实例变量或全局变量进行共享。

例如，在单线程应用程序中可能会维持一个全局的数据库连接，并在程序启动时初始化这个连接对象，从而避免在调用每个方法时都要传递一个Connection对象。由于JDBC的连接对象不一定是线程安全的，因此，当多线程应用程序在没有协同的情况下使用全局变量时，就不是线程安全的。通过将JDBC的连接保存到ThreadLocal对象中，每个线程都会拥有自己的连接，如下所示：

```java
private static ThreadLocal<Connection> connectionHolder 
    = new ThreadLocal<Connection>()
    {
        public Connection initialValue()
        {
            return DriverManager.getConnection(DB_URL);
        }
    };

    public static Connection getConnection()
    {
        return conncetionHolder.get();
    }
```

当某个频繁执行的操作需要一个临时对象，例如一个缓冲区，而同时又希望避免在每次执行时都重新分配该临时对象，就可以使用这项技术。

当某个线程初次调用ThreadLocal.get方法时，就会调用initialValue来获取初始值。从概念上看，你可以将ThreadLocal<T>视为包含了Map< Thread,T>对象，其中保存了特定于该线程的值，当ThreadLocal的实现并非如此。这些特定于线程的值保存在Thread对象中，当线程终止后，这些值会作为垃圾回收。

假设你需要将一个单线程应用程序移植到多程序环境中，通过将共享的全局变量转换为ThreadLocal（如果全局变量的语义允许），可以维护线程安全性。然而，如果将应用程序范围内的转换为线程局部的缓存，就不会有太大作用<!--这是因为缓存本来就是全局唯一，供所有的线程共享，如果转换为ThreadLocal，那么每个线程都会有一个缓存，缓存就失去原本的意义-->。

开发人员经常滥用ThreadLocal，例如将所有全局变量都作为ThreadLocal对象，或者作为一种“隐藏”方法参数的手段。ThreadLocal变量类似于全局变量，他能降低代码的可重用性，并在类之间引入隐含的耦合性，因此在使用时要格外小心。

## 不可变性

满足同步需求的另一种方法是使用不可变对象。

创建后状态不能被修改的对象叫做不可变对象。不可变对象天生就是线程安全。它们常量域是在构造函数中创建的。既然它们的状态无法被修改，这些常量永远不会变。所以不可变对象永远是线程安全的。

> 不可变对象一定是线程安全的。

不可变性并不简单地等于将对象中的所有域都声明为final类型，所有域都是final类型的对象仍然可能是可变的，因为final域可以获得一个到可变对象的引用。

> 只有满足如下状态，一个对象才是不可变的：
> 1、 对象的状态不能在创建后再被修改；
> 2、 对象的所有域都是final类型；
> 3、 对象被正确创建（创建期间没有发生this引用逸出）。

在不可变对象的内部，同样可以使用可变性对象来管理它们的状态，如下面代码，虽然域stooges是可变的，但它满足了以上三点，所以是一个不可变对象：

```java
@Immutable//不可变对象可以基于可变对象来实现
public final class ThreeStooges {
    private final Set<String> stooges = new HashSet<String>();
    public ThreeStooges() {
        stooges.add("Moe");
    }
    public boolean isStooge(String name) {
        return stooges.contains(name);
    }
}
```

### final域

final类型的域时不能修改的（但如果final域所引用的对象是可变的，那么这些被引用的对象是可以修改的<!--毕竟final类型的域，其实是个地址，这个地址是不可变的，但是地址指向的对象是可变的-->）。然而，在Java内存模型中，final域还有着特殊的语义。final域能确保初始化过程的安全性 <!--想想类加载的过程-->，从而可以不受限制地访问不可变对象，并在共享这些对象时无须同步。

仅包含一个或两个可变状态的“基本不可变”对象仍然比包含多个可变状态的对象简单。通过将域声明为final类型的，也相当于告诉维护人员这些域时不会变化的。

> 正如“除非需要更高的可见性，否则应将所有的域声明为私有域是一个良好的编程习惯，”除非需要某个域是可变的，否则应将其声明为final域“也是一个良好的编程习惯。

### 示例：使用Volatile类型来发布不可变对象

尽管原子引用自身是线程安全的，不过UnsafeCachingFactorizer中存在竞争条件 <!--第二章的复合操作"读取-修改-写入"，违背了状态一致性，读取了过期数据-->，当前线程在执行到A 与 B之间或者C 与 D之间，都有可能切换到其他线程，从而造成错误的结果。

```java
//没有正确原子化的Servlet试图缓存它的最新结果。
@NotThreadSafe
public class UnsafeCachingFactorizer implements Servlet {
     private final AtomicReference<BigInteger> lastNumber
         = new AtomicReference<BigInteger>();//缓存最后一次客户请求因式分解的数
     private final AtomicReference<BigInteger[]>  lastFactors
         = new AtomicReference<BigInteger[]>();//缓存最后一次客户请求因式分解的结果

     public void service(ServletRequest req, ServletResponse resp) {
         BigInteger i = extractFromRequest(req);
       	 //先检查后执行，必须保证lastNumber不是过期数据
         if (i.equals(lastNumber.get()))//A
             encodeIntoResponse(resp,  lastFactors.get() );//B
         else {
             BigInteger[] factors = factor(i);
           	 //lastNumber和lastFactors他俩必须维持状态一致性
             lastNumber.set(i);//C
             lastFactors.set(factors);//D
             encodeIntoResponse(resp, factors);
         }
     }
}
```

[如果上面的A与B这个复合操作操作、以及C与D这个复合操作如果是原子性的，那么将不会出现线程安全性问题。如果为这两组操作创建一个不可变的类，即使在不使用同步的情况也能解决安全共享问题。]() <!--通过创建不可变类，对复合操作进行原子性，good idea--> 

下面就为UnsafeCachingFactorizer创建一个OneValueCache类，对以上操作进行了封装，它是一个不可变对象，进（构造时传进的参数）出（使用时）都对状态进行了拷贝。因为BigInteger是不可变的，所以直接使用了Arrays.copyOf来进行拷贝了，如果状态所指引的对象不是不可变对象时，就要不能使用这项技术了，因为外界可以对这些状态所指引的对象进行修改，如果这样只能使用new或深度克隆技术来进行拷贝了。

> 每当需要对一组相关数据以原子方式执行某个操作时，就可以考虑创建一个不可变的类来包含这些数据。
>
> <!--nb-->

```java
@Immutable
class OneValueCache {
    private final BigInteger lastNumber;
    private final BigInteger[] lastFactors;

    public OneValueCache(BigInteger i,
                         BigInteger[] factors) {
        lastNumber  = i;
        lastFactors = Arrays.copyOf(factors, factors.length);
    }

    public BigInteger[] getFactors(BigInteger i) {
        if (lastNumber == null || !lastNumber.equals(i))
            return null;
        else
            return Arrays.copyOf(lastFactors, lastFactors.length);
    }
}
```

对于在访问和更新多个相关变量时出现的竞争条件问题，可以通过将这些变量全部保存在一个不可变对象中来消除。

- 如果是一个可变的对象，那么就必须使用锁来确保原子性。

- 如果是一个不可变对象，那么当线程获得了对该对象的引用后，就不必担心另一个线程会修改对象的状态。 <!--这段话很有意思，需要结合servlet背景并且联系前边的知识点：因为servlet对每一次服务请求都是会被new一次，所以每个请求都会有自己的--> 

- 如果要更新这些变量，那么可以创建一个新的容器对象，但其他使用原有对象的线程仍然会看到对象处于一致的状态。

当一个线程将volatile类型的cache设置为引用一个新的OndeValueCache时，其实线程就会立即看到新缓存的数据。

```java
@ThreadSafe
public class VolatileCachedFactorizer implements Servlet {
    private volatile OneValueCache cache =
        new OneValueCache(null, null);//使用volatile安全发布
  
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = cache.getFactors(i);
        if (factors == null) {
            factors = factor(i);
            //由于cache为volatile，所以最新值立即能让其它线程可见
            cache = new OneValueCache(i, factors);
        }
        encodeIntoResponse(resp, factors);
    }
}
```

与cache相关的操作不会相互干扰，因为OneValueCache是不可变的，并且在每条相应的代码路径中只会访问它一次 <!--这段话很有意思，看我上边的注释--> 。通过使用包含过个状态变量的容器对象来维护不可变条件，并使用一个volatile类型的引用来确保可见性，使得Volatile Cached Factorizer在没有显式地使用锁的情况下仍然是线程安全的。



## 安全发布

到目前为止，我们重点讨论的是如何确保对象不被发布，比如让对象封闭在线程或者另一个对象内部。当然，我们希望多个线程间安全地进行共享数据。下面程序中简单地将对象的引用存储到public域中，这不足以安全地发布它：

```java
// 不安全的发布：在没有适当的同步情况下就发布对象
public Holder holder;
public void initialize(){
    holder = new Holder(42);
}
```

由于存在可见性问题，其他线程看到的Holder对象将处于不一致的状态，即便在该对象的构造函数中已经正确地构建了不变性条件。这种不正确的发布导致其他线程看到尚未创建完成的对象。

### 3.5.1 不正确的发布：正确的对象被破坏

```java
public class Holder {
    private int n;

    public Holder(int n) { this.n = n; }

    public void assertSanity() {
        if (n != n)//在不正确的发布中，是很有可能出现不等
            throw new AssertionError("This statement is false.");
    }
}
```

上面程序的问题是由于对象的可见性问题引起的（<!--问题不再于Holder类本身，而在于Holder类未被正确的发布，如果把n声明为final，那么Holder将不可变，满足正确发布的要求，具体看3.5.2节。现在最根本的原因是由于JVM运行时重排序引起的-->），发布的对象可能还处于构造期间，所以是不稳定的。因为没有同步来确保Holder对其他线程可见，所以我们称Holder是“非正确发布”。

由于上面 n != n 会从主存中两次读取，这有可能从这两次读操作间切换到其他线程，这就有可能出 n!=n奇怪的问题。

### 3.5.2 不可变对象与初始化安全性

Java内存模型为共享不可变对象提供了特殊的初始化安全性的保证，即对象在完全初始化之后才能被外界引用，所以只要是不可变对象，一旦构建完成，就可以安全地发布了。

即使某个对象的引用对其他线程是可见的，也并不意味着对象状态对于使用该对象的线程来说一定是可见的。为了确保对象状态能呈现出一致的视图，就必须使用同步。

即使发布对象引用时没有使用同步，不可变对象仍然可以被安全地访问（注，只能保证一旦看到的对象就是完整的，在没有使用同步的情况下是不能保证对象引用的可见性，所以不可变对象只能保证初始化完后的就处于稳定状态）。**为了获得这种初始化安全性的保证上，应该满足所有不可变性的条件**：<!--不可修改的状态、所有域都是final类型的、正确的构造-->。（如果上面的Holder是不可变的，那么即使Holder没有正确的发布，assertSanity也不会抛出AssertionError。）

> 任何线程都可以在不需要额外同步的情况下安全地访问不可变对象，即使在发布这些对象时没有使用同步。

这个保证还会延伸到一个正确创建的对象中所有final类型域的值。final域可以在没有额外的同步情况下被安全地访问（因为只要构造器一旦调用完毕，则final域的也会随之初始化完并可见），然而，如果final域指向可变对象，那么访问这些对象的状态时仍然需要同步的。

### 3.5.3 安全发布的常用模式（可变对象的安全发布）

如果一个对象不是不可变的，它就必须要被安全的发布，通常发布线程与消费线程都必须同步。我们要确保消费线程能够看到处于发布当时的对象状态。

> 为了安全地发布一个可变对象，对象的引用以及对象的状态必须同时对其他线程可见。一个正确构造的对象可以通过以下方式来安全发布：
> 在静态初始化函数中初始化一个对象引用；static{}
> 将对象的引用保存到volatile类型的域或者AtomicReferance对象中
> 将对象的引用保存到某个正确构造对象的final类型域中
> 将对象的引用保存到一个由锁保护的域中

线程安全中的容器提供了线程安全保证（即变向地将对象置于了同步器中进行访问），正是遵守了上述最后一条要求。

线程安全库中的容器类提供了以下的安全发布保证：

- 通过将一个键或者值放入Hashtable、synchronizedMap或者ConcurrentMap中，可以安全地将它发布给任何从这些容器中访问它的线程（无论是直接访问还是通过迭代器访问）。
- 通过将某个元素放入Vector、CopyOnWriteArrayList、CopyOnWriteArraySet、synchronizedList或synchronizedSet中，可以将该元素安全地发布到任何从这些容器中访问该元素的线程。
- 通过将某个元素放入BlockingQueue或者ConcurrentLinkedQueue中，可以将该元素安全地发布到任何从这些队列中访问该元素的线程。

类库中的其他数据传递机制（例如Future和Exchanger）同样能实现安全发布。

要发布一个静态构造的对象，最简单和最安全的方式是使用静态的初始化器：

```
public static Holder holder = new Holder(42);1
```

静态初始化器由JVM在类的初始化阶段执行。由于在JVM内部存在着同步机制，因此通过这种方式初始化的任何对象都可以被安全地发布。

**如果对象在创建后被修改，那么安全发布仅仅可以保证“发布当时”状态的可见性。不仅仅在发布对象时需要同步，而且在对象发布后修改了对象状态又要让其他线程可见，则也需要对每次状态的访问进行同步。为了安全地共享可变对象，可变对象必须被安全发布，同时对状态的访问需要同步化。**

### 3.5.4 事实不可变对象

对于对象在发布后不会被修改，那么对于其他在没有额外同步的情况下安全地访问这些对象的线程来说，安全发布时足够的。

如果对象从技术上来看时可变的，但其状态在发布后不会再改变，那么把这种对象称为“事实不可变对象”。

用事实不可变对象可以简化开发，并且由于减少了同步的使用，还会提高性能。

> 在没有额外同步的情况下，任何线程都可以安全地使用被安全发布的事实不可变对象。

比如，Date自身是可变的（这也许是类库设计的一个错误），但是如果你把它当作不可变对象来使用就可以忽略锁。否则，每当Date被跨线程共享时，都要用锁确保安全。假设你正在维护一个Map，它存储了每位用户的最近登录时间：

```java
public Map<String, Date> lastLogin =
Collections.synchronizedMap(new HashMap<String, Date>());
```

如果Date值在转入Map中后就不会改变，那么，synchronizedMap中同步的实现就足以将Date安全地发布，并且访问这些Date值时就不再需要额外的同步。

### 3.5.5 可变对象–>对象（不可变/可变/高效）的发布约束

如果对象在构造后可以修改，那么安全发布只能确保“发布当时”状态的可见性。对于可变对象，不仅在发布对象时需要使用同步，而且在每次对象访问时同样需要使用同步来确保后续操作的可见性。**要安全地共享可变对象，这些对象就必须被安全地发布，并且必须是线程安全的活着由某个锁保护起来。**

> 对象的发布需求取决于它的可变性：
> \* 不可变对象可以通过任意机制来发布
> \* 事实不可变对象必须通过安全方式来发布
> \* 可变对象必须通过安全方式来发布，并且必须是线程安全的或者由某个锁保护起来

### 3.5.6 安全地共享对象

当获得对象的一个引用时，你需要知道在这个引用上可以执行哪些操作。在使用它之前是否需要获得一个锁？是否可以修改它的状态，或者只能读取它？许多错误都是由于没有理解共享对象的这些“既定规则”而导致。当发布一个对象时，必须明确地说明对象的访问方式。

> 在并发程序中使用和共享对象时，可以使用一些使用的策略，包括：
> **线程封闭**。线程封闭的对象只能由一个线程拥有，对象被封闭在该线程中，并且只能由这个线程修改。
> **只读共享。**在没有额外同步的情况下，共享的只读对象可以由多个线程并发访问，但任何线程都不能修改它。共享的只读对象包括不可变对象和事实不可变对象。
> **线程安全共享。**线程安全的对象在其内部实现同步，因此多个线程可以通过对象的公有接口来进行访问而不需要进一步的同步。
> **保护对象。**被保护的对象只能通过持有特定的锁来访问。保护对象包括封装在其他线程安全对象中的对象，以及已发布的并且由某个特定锁保护的对象。



