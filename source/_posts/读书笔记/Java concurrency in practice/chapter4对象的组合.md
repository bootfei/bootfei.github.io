---
title: chapter4对象的组合
date: 2020-12-17 11:57:11
tags: [并发]
---

# 对象的组合

我们并不希望每一次内存访问都进行分析以确保程序时线程安全的，而是希望将一些现有的线程安全组件组合为更大规模的组件或程序。本章将介绍一些组合模式，这些模式能够将一个类更容易成为线程安全的，并且在维护这些类时不会无意中破坏类的安全性保证。

## 设计线程安全的类

通过使用封装技术，可以使得在不对整个程序进行分析的情况下就可以判断一个类是否时线程安全的。

> 在设计线程安全类的过程中，需要包含以下三个基本要素：
> \* 找出构成对象状态的所有变量。
> \* 找出约束状态变量的不变性条件。
> \* 建立对象状态的并发访问管理策略。

要分析对象的状态，首先从对象的域开始。

> 如果在对象的域中都是基本类型的变量，那么这些域将构成对象的全部状态。
>
> 如果在对象的域中引用了其他对象，那么该对象的状态就包含被引用对象的域。

同步策略定义了如何在不违背对象不变性条件或后验条件的情况下对其状态的访问操作进行协同。同步策略规定了如何将不可变性、线程封闭与加锁机制等结合起来以维护线程的安全性，并且还规定了哪些变量由哪些锁来保护。要确保开发人员可以对这个类进行分析与维护，就必须将同步策略写为正式文档。

### 收集同步需求

要确保类的线程安全性，就需要确保它的不变性条件不会在并发访问的情况下被破坏，这就需要对其状态进行推断。对象与变量都有一个状态空间，即所有可能的取值。状态空间越小，就越容易判断线程的状态。final类型的域使用的越多，就能简化对象可能状态的分析过程。（不可变对象只有唯一的状态）

许多类中定义了一些不可变条件，拥有判断状态是有效的还是无效的。long类型的变量，其状态空间为从Long.MIN_VALUE到Long.MAX_VALUE，但Counter中value取值范围存在限制，即不能是负值。

在操作中还会包含一些后验条件来判断状态迁移是否是有效的。如果Counter的当前状态是17，那么下一个有效状态只能是18。**当下一个状态需要依赖当前状态时，这个操作就必须是一个复合操作。**并非所有的操作都会在状态转换上施加限制，例如，当更新一个保存当前温度的变量时，该变量之前的状态并不会影响计算结果。

[由于不变性条件以及后验条件在状态及状态转换上施加了各种约束]()，因此就需要额外的同步与封装。如果某些状态是无效的，那么必须对底层的状态变量进行封装，否则客户端代码可能会使对象处于无效状态。如果在某个操作中存在无效的状态转换，那么该操作必须是原子的。如果没有施加这种约束，那么就可以放宽封装性或序列化需求，以便获得更高的灵活性或性能。

在类中也可以包含同时约束多个状态变量的不变性条件。在一个表示数值范围的类中可以包含两个状态变量，分别表示范围的上界和下界。这些变量必须遵循的约束是，下界值应该小于或等于上界值。类似于这种包含多个变量的不变性条件将带来原子性需求：这些相关的变量必须在单个原子操作中 <!--复合操作--> 进行读取或更新。不能首先更新一个变量，然后释放锁并再次获得锁，然后再更新其他的变量。因为在释放锁后，可能会使对象处于无效状态。如果在一个不变性条件中包含多个变量，那么在执行任何访问相关变量的操作时，都必须持有保护这些变量的锁。

> 如果不了解对象的不变性条件与后验条件，那么就不能确保线程安全性。要满足在状态变量的有效值或状态转换上的各种约束条件，就需要借助于原子性与封装性。

### 依赖状态的操作

类的不变性条件与后验条件约束了在对象上有哪些状态和转换是有效的。在某些对象的方法中还包含一些基于状态的先验条件。例如，不能够从空队列中移除一个元素，在输出元素前，队列必须处于“非空的”状态。如果在某个操作中包含有基于状态的先验条件，那么这个操作就称为依赖状态的操作。

在单线程程序中，如果某个操作无法满足先验条件，那么就只能失败。但在并发程序中，先验条件可能会由于其他线程执行的操作而变为真 <!--这就是Bug了--> 。[在并发程序中要一直等到先验条件为真，然后再执行该操作]()。

在Java中，等待某个条件为真的各种内置机制，（包括等待和通知等机制）都与内置加锁机制紧密关联，要想正确地使用他们并不容易。要想实现某个等待先验条件为真时才执行的操作，一种更简单的方法是通过现有库中的类（例如阻塞队列【BlockingQueue】或信号量【Semaphore】）来实现依赖状态的行为。

### 状态的所有权

如果以某个对象为根节点构造一张对象图，那么该对象的状态将是对象图中所有对象包含的域的一个子集。

在定义哪些变量将构成对象的状态时，只考虑对象拥有的数据。如果分配并填充了一个HashMap对象，那么就相当于创建多个对象：HashMap对象，在HashMap对象中包含的多个对象，以及在Map.Entry中可能包含的内部对象。HashMap对象的逻辑状态包括所有的Map.Entry对象以及内部对象，即使这些对象都是一些独立的对象。

所有权与封装性总是相互关联的：对象封装它拥有的状态，反之也成立，对它封装的状态拥有所有权。状态变量的所有者将决定采用何种加锁协议来维持变量状态的完整性。所有权意味着控制权。然而，如果发布了某个可变对象的引用，那么就不再拥有独占的控制权，最多是“共享控制权”。对于从构造函数或者从方法中传递进来的对象，类通常并不拥有这些对象，除非这些方法是被专门设计为转移传递进来的对象的所有权（例如，同步容器封装器的工厂方法）。

容器类通常变现出一种“所有权分离”的形式，其中容器类拥有其自身的状态，而客户代码则拥有容器中各个对象的状态。Servlet框架中的ServletContext就是其中一个示例。ServletContext为Servlet提供了类似于Map形式的对象容器服务，在ServletContext可以通过名称来注册（setAttribute）或获取（getAttribute）应用程序对象。由Servlet容器实现的ServletContext对象必须是线程安全的，因为它肯定会被多个线程同时访问。当调用setAttribute和getAttribute时，Servlet不需要使用同步，但当使用保存在ServletContext中的对象时，则可能需要使用同步。这些对象由应用程序 <!--客户代码--> 拥有，Servlet容器只是替应用程序保管它们。与所有共享对象一样，它们必须安全地被共享。为了防止多个线程在并发访问同一个对象时产生的相互干扰，这些对象要么是线程安全的对象，要么是事实不可变的对象，或者由锁来保护的对象。

## 实例封闭(我上章说的对象封闭)

如果某对象不是线程安全的，那么可以通过多种技术使其在多线程程序中安全地使用。你可以确保该对象只能由单个线程访问（线程封闭），或者一个锁来保护该对象的所有访问<!--复习第2章-->

封装简化了线程安全类的实现过程，它提供了**实例封闭机制**。当一个对象被封闭到另一个对象中时，能够访问被封闭对象的所有代码路径都是已知。与对象可以由整个程序访问的情况相比，更易于对代码进行分析。通过封闭机制与合适的加锁策略结合起来，可以确保以线程安全的方式来使用非线程安全的对象。

> 将数据封装在对象内部，可以将数据的访问限制在对象的方法上，从而更容易确保线程在访问数据时总能持有正确的锁。

被封闭对象一定不能超出既定的作用域。对象可以封闭在类的一个实例（例如作为类的一个私有成员）中，或者封闭在某个作用域内（例如作为一个局部变量），再或者封闭在线程内（例如在某个线程中将对象从一个方法传递到另一个方法，而不是在多个线程之间共享该对象）。

通过封闭机制来确保线程安全，通过封闭与加锁等机制使一个类成为线程安全的（即使这个类的状态变量并不是线程安全的）。

```java
//通过封闭机制来确保线程安全
public class PersonSet{
    private final Set<Person> mySet = new HashSet<Person>();

    public synchronized void addPerson(Person p){
        mySet.add(p);
    }

    public synchronized void containsPerson(Person p){
        return mySet.contains(p);
    }
}
```

PersonSet的状态由HashSet来管理，而HashSet并非线程安全的<!--PersonSet是类的对象，mySet是其中一个域，那么PersonSet的状态就包含mySet以及mySet引用的Person对象，复习开头的定义-->。但由于mySet是私有的并且没有逸出，因此HashSet被封闭在PersonSet中。唯一能访问mySet的代码路径是addPerson与containsPerson，在执行它们时都要获得PersonSet上的锁。PersonSet的状态完全由它的内置锁保护满足<!--第二章的加锁约定-->，因此PersonSet是线程安全的类。

但如果Person类是可变的，那么在访问从PersonSet中获得的Person对象时<!--比如public Person getPerson()-->，还需要额外的同步。要想安全地使用Person对象，可以使Person对象成为一个线程安全类。也可以使用锁来保护Person对象，并确保所有客户端代码在访问Person对象之前都已经获得正确的锁。

实例封闭时构建线程安全类的一个最简单方式，它使得在锁策略的选择上拥有了更多地灵活性。在PersonSet中可以使用内置锁来保护它的状态，对于其他形式的锁只要自始至终都使用同一个锁，就可以保护状态。**实例封闭还使得不同的状态可以由不同的锁来保护**。

在Java平台的类库中还有很多实例封闭的示例，其中有些类的唯一用途就是将非线程安全的类转化为线程安全的类。一些基本的容器类例如ArrayList不是线程安全的，但类库提供了包装器工厂方法，例如**Collections.synchronizedList及其类似方法，使得这些非线程安全的类可以在多线程环境中安全地使用。这些工厂方法通过”装饰器Decorator”模式将容器封装在一个同步的容器对象上，而包装器能将接口中的每个方法都实现为同步方法，并将调用请求转发到底层的容器对象上。只要包装器对象拥有对底层容器对象的唯一引用（即把底层容器对象封闭在包装器中），那么它就是线程安全的。在这引起方法的Javadoc中指出，对底层容器对象的所有访问必须通过包装器来进行。**

如果将一个本该被封闭的对象发布出去，那么也能破坏封闭性。如果一个对象本应该封闭在特定的作用域内，那么让该对象逸出作用域就是一个错误。当发布其他对象时，例如迭代器或内部的类实例，可能会间接地发布被封闭对象，同样会使被封闭对象逸出。

> 封闭机制更易于构造线程安全的类，因为当封闭类的状态时，在分析类的线程安全性时就无须检查整个程序。

### Java监视器模式

进入和退出同步代码的字节指令也称为monitorenter和monitorexit，而Java的内置锁也称为监视器锁或监视器。

遵循Java监视器模式的对象会把对象的所有可变状态都封装起来，并由对象自己的内置锁来保护。

Java监视器模式的主要优势就在于它的简单性，11章介绍通过更细粒度的加锁策略来提高可伸缩性。

**Java监视器模式仅仅是编写代码的约定，对于任何一种锁对象，只要自始至终都使用该锁对象**，都可以用来保护对象的状态。如下程序给出了如何使用私有锁来保护状态。

```java
//通过一个私有锁来保护状态
public class PrivateLock{
    private final Object myLock = new Object();
		@GuardedBy("myLock")Widget widget;
    void someMethod(){
        synchronized(myLock){
            // 访问或修改Widget的状态
        }
    }
}
```

私有的锁对象可以将锁封装起来，使客户代码无法得到锁，但客户代码可以通过公有方法来访问锁，以便（正确或者不正确）参与到它的同步策略中。

### 示例：车辆追踪

　　以下程序清单中，我们看一个示例： 一个用于调度车辆的“车辆追踪器”。首先使用监视器模式来构建车辆追踪器，然后尝试放宽某些封装性需求同时又保持线程安全性。

```java
@ThreadSafe
public class MonitorVehicleTracker {
    private final Map<String ,MutablePoint> locations;
    public MonitorVehicleTracker(Map<String ,MutablePoint> locations) {
        this.locations = deepCopy(locations);   //返回拷贝信息
    }

    public synchronized Map<String, MutablePoint> getLocations() {
        return deepCopy(locations); //返回拷贝信息
    }

    public synchronized MutablePoint getLocation(String id) {
        MutablePoint lo = locations.get(id);
        return lo == null ? null : new MutablePoint(lo);    //返回拷贝信息
    }

    public synchronized void setLocations(String id, int x, int y) {
        MutablePoint lo = locations.get(id);
        if (lo == null) {
            throw new IllegalArgumentException("");
        }
        lo.x = x;
        lo.y = y;
    }

    private static Map<String, MutablePoint> deepCopy(Map<String, MutablePoint> locations) {
        Map<String, MutablePoint> result = new HashMap<String, MutablePoint>();
        for (String id : locations.keySet()) {
            result.put(id, new MutablePoint(locations.get(id)));
        }
        return Collections.unmodifiableMap(result);
    }
}
```



```java
@NotThreadSafe【不要这么做】
public class MutablePoint {　　　　　　　
    public int x, y;
    public MutablePoint() {
        x = 0; y = 0;
    }
    public MutablePoint(MutablePoint p) {
        this.x = p.x;
        this.y = p.y;
    }
}
```

　　虽然类 MutablePoint 不是线程安全的，但追踪器类时线程安全的，<!--它所包含的 Map 对象和可变的 Point 对象都未曾发布-->。当需要返回车辆的位置时，通过 MutablePoint 拷贝构造函数或者 deepCopy 方法来复制正确的值，从而生成一个新的Map 对象 <!--更准确来说，新的句柄和新的对象-->，并且该对象中的值与原有 Map 对象中的 key 值和 value 值都相同。

　　在某种程度上，这种实现方式是通过再返回客户代码之前复制可变的数据来维持线程安全性的。通常情况下，这并不存在性能问题，但在车辆容器非常大的情况下将极大地降低性能。<!--这么多数据副本，内存不够了-->

## 线程安全性的委托(不依赖锁，依赖内部类是线程安全的)

大多数对象都是组合对象。当从头开始构建一个类，或者将多个非线程安全的类组合为一个类时，Java监视器模式是非常有用的。如果类中的各个组件都已经是线程安全的，会是什么情况？是否需要再增加一个额外的线程安全层？答案是“视情况而定”。在某些情况下，通过多个线程安全类组合而成的类是线程安全的，而在某些情况下，仅仅是好的开端。

### 示例：基于委托的车辆追踪器

```java
@Immutable
public class Point {
    public final int x, y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}

//将线程安全委托给ConcurrentMap
@ThreadSafe
public class DelegatingVehicleTracker {
    private final ConcurrentMap<String, Point> locations;
    private final Map<String, Point> unmodifiableMap;

    public DelegatingVehicleTracker(Map<String, Point> points) {
        locations = new ConcurrentHashMap<String, Point>(points);
        unmodifiableMap = Collections.unmodifiableMap(locations);
    }

    public Map<String, Point> getLocations() {
        return unmodifiableMap;
    }

    public Point getLocation(String id) {
        return locations.get(id);
    }

    public void setLocation(String id, int x, int y) {
        if (locations.replace(id, new Point(x, y)) == null)
            throw new IllegalArgumentException("invalid vehicle name: " + id);
    }

    // Alternate version of getLocations (Listing 4.8)
    public Map<String, Point> getLocationsAsStatic() {
        return Collections.unmodifiableMap(
                new HashMap<String, Point>(locations));
    }
}
```

由于Point类使不可变的，所以它是线程安全的。[不可变的值可以被自由地共享与发布]()，[因此在返回location时不需要复制。]()DelegatingVehicleTracker没有使用任何显示的同步，所有对状态的访问都由ConcurrentHashMap来管理<!--unmodifiableMap是线程安全的，不影响其状态，所以不考虑了-->，而且Map所有的key和value都是不可变的。

如果使用最初的MutablePoint类而不是Point类，就会破坏封装性，因为Map<String, Point> getLocations() 会发布一个指向可变状态的引用，而这个引用不是线程安全的。

> 在使用监视器模式的车辆追踪器中返回的是车辆位置的快照，而在使用委托车辆追踪器中返回的是一个不可修改但却实时的车辆位置试图。这意味着，如果线程A调用getLocations，而线程B随后修改了某些点的位置，那么在返回给线程A的Map中将反映出这些变化。这可能是优点(更新的数据)，可能是缺点（可能导致不一致的车辆位置视图），具体情况取决于你的需求。

如果委托的车辆追踪器也系统得到车辆位置的快照，那么getLocations可以返回对locations这个Map对象的一个浅拷贝。由于Map的内容是不可变的，因此只需复制Map的结构，而不用复制它的内容。

### 独立的状态变量

可以将线程安全性委托给多个状态变量，**只要这些变量是彼此独立的**，<!--即组合而成的类，并不会在其包含的多个状态变量上增加任何不变性条件，或者说约束条件-->。

```java
public class VisualComponent {
    private final List<KeyListener> keyListeners
            = new CopyOnWriteArrayList<KeyListener>();
    private final List<MouseListener> mouseListeners
            = new CopyOnWriteArrayList<MouseListener>();

    public void addKeyListener(KeyListener listener) {
        keyListeners.add(listener);
    }

    public void addMouseListener(MouseListener listener) {
        mouseListeners.add(listener);
    }

    public void removeKeyListener(KeyListener listener) {
        keyListeners.remove(listener);
    }

    public void removeMouseListener(MouseListener listener) {
        mouseListeners.remove(listener);
    }
}
```

VisualComponent使用CopyOnWriteArrayList来保存各个监听器列表。它是一个线程安全的链表，特别适用于管理监听器列表。每个列表都是线程安全的，由于各个状态之间不存在耦合关系，<!--比如key和mouse就是彼此完全独立的--> 因此VisualComponent可以将它的线程安全性委托给mouseListeners和keyListeners等对象。

### 当委托失效时

大多数组合对象都不会像VisualComponent这样简单：**在它们的状态变量之间存在着某些不变性条件**。NumberRange使用了两个AtomicInteger来管理状态，并且含有一个约束条件，即第一个数值要小于或等于第二个数值。

```java
public class NumberRange {
    // INVARIANT: lower <= upper
    private final AtomicInteger lower = new AtomicInteger(0);
    private final AtomicInteger upper = new AtomicInteger(0);

    public void setLower(int i) {
        // Warning -- unsafe check-then-act
        if (i > upper.get())
            throw new IllegalArgumentException("can't set lower to " + i + " > upper");
        lower.set(i);
    }

    public void setUpper(int i) {
        // Warning -- unsafe check-then-act
        if (i < lower.get())
            throw new IllegalArgumentException("can't set upper to " + i + " < lower");
        upper.set(i);
    }

    public boolean isInRange(int i) {
        return (i >= lower.get() && i <= upper.get());
    }
}
```

NumberRange 不是线程安全的<!--NumberRange是含有隐藏的不变性条件的，因为boolean isInRange(int i)是依赖于lower和upper两个变量，所以两个变量有约束条件-->，没有维持对下界和上界进行约束的不变性条件。假设取值范围在（0， 10），如果一个线程调用 setLower（5），而另一个线程调用 setUpper（4），那么在一些错误的执行时序中，比如下图所示

| ThreadA                                | ThreadB                                |
| :------------------------------------- | -------------------------------------- |
| Line6: setLower（5）                   |                                        |
| Line8: i > upper.get() = 10 通过了检查 |                                        |
|                                        | Line13: setUpper（4）                  |
|                                        | LIne15: i < lower.get() = 0 通过了检查 |
| Line10: lower.set(i);                  |                                        |
|                                        | LIne17: upper.set(i);                  |

这两个调用都通过了检查，并且都设置成功。因此，虽然 AtomicInteger 是线程安全的，但经过组合得到的类却不是线程安全的。**由于状态变量lower和upper不是彼此独立的**<!--前面一节的代码中，状态变量都是彼此独立的而且线程安全，所以类的线程安全性可以委托给线程安全状态变量-->，**因此NumberRange不能将线程安全性委托给他的线程安全状态变量**。

如果某个类含有复合操作，例如NumberRange，那么仅靠委托并不足以实现线程安全性。在这种情况下，这个类必须提供自己的加锁机制以保证这些复合操作都是原子操作，除非整个复合操作都可以委托给状态变量。

> 如果一个类是由多个独立且线程安全的状态变量组成，并且在所有的操作中都不包含无效状态转换，那么可以将线程安全性委托给底层的状态变量。

### 发布底层的状态变量

当把线程安全性委托给某个对象的底层状态变量时，在什么条件下才可以发布这些变量从而使其他类能修改它们？

- 答案仍然是取决于在类中对这些变量施加了哪些不变性条件。
  - 虽然Counter中的value域可以为任何整数值，但Counter施加的约束条件是只能取正整数，此外递增操作同样约束了下一个状态的有效取值范围。如果将value声明为一个公有域，那么客户代码可以将它修改为一个无效值，因此发布value会导致这个类出错 <!--当前counter依赖于上一次counter结果，有约束条件or不变性条件--> 。
  - 另一方面，如果某个变量表示的时当前温度或者最近登录用户的ID，那么即使另一个类在某个时刻修改了这个值，也不会破坏任何不变性条件，因此发布这个变量也是可以接受的。<!--温度和ID没有约束条件-->

> 如果一个状态变量时线程安全的，并且没有任何不变性条件来约束它的值，在变量的操作上也不存在任何不允许的状态转换，那么就可以安全地发布这个变量。<!--这就是正确地运用线程安全性的委托-->

例如，发布VisualComponent中的mouseListeners或keyListeners等变量就是安全的。由于VisutalComponent并没有在其监听器链表的合法状态上施加任何约束，因此这些域可以声明为共有域或者发布，而不会破坏线程安全性。

### 示例：发布状态的车辆追踪器

我们来构造车辆追踪器的另一个版本，并在这个版本中发布底层的可变状态。我们需要修接口以适应这种变化，即使用可变且线程安全的 Point 类。<!--这个实例就是呼应了4.3.4内容，发布底层对象的状态变量。注意和4.3.1示例的区别，4.3.1示例的底层对象是没有发布对象的，而且是不可变的。现在是发布+可变-->

```java
@ThreadSafe
public class SafePoint {
    @GuardedBy("this") private int x, y;

    private SafePoint(int[] a) {
        this(a[0], a[1]);
    }

    public SafePoint(SafePoint p) {
        this(p.get());
    }

    public SafePoint(int x, int y) {
        this.set(x, y);
    }

    public synchronized int[] get() {
        return new int[]{x, y};
    }

    public synchronized void set(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
@ThreadSafe
public class PublishingVehicleTracker {
    private final Map<String, SafePoint> locations;
    private final Map<String, SafePoint> unmodifiableMap;

    public PublishingVehicleTracker(Map<String, SafePoint> locations) {
        this.locations = new ConcurrentHashMap<String, SafePoint>(locations);
        this.unmodifiableMap = Collections.unmodifiableMap(this.locations);
    }

    public Map<String, SafePoint> getLocations() {
        return unmodifiableMap;
    }

    public SafePoint getLocation(String id) {
        return locations.get(id);
    }

    public void setLocation(String id, int x, int y) {
        if (!locations.containsKey(id))
            throw new IllegalArgumentException("invalid vehicle name: " + id);
        locations.get(id).set(x, y);
    }
}
```

## 在现有的线程安全类中添加功能

有的时候，现有的类只能支持大部分的操作，此时就需要在不破坏线程安全性的情况下添加一个新的操作。

要添加一个新的原子操作，最安全的做法时修改原始的类，但这通常无法做到，因为你可能无法访问或修改类的源代码。要想修改原始的类，就需要理解代码中的同步策略。这样增加的功能才能与原有的设计保持一致。如果直接将新方法添加到类中，那么意味着实现同步策略的所有代码仍然处于一个源代码文件中，从而更容易理解与维护。

另一种方法是扩展这个类，假定在设计这个类时考虑了可扩展性。但是并非所有的类都像Vector那样将状态向子类公开，因此也就不适合采用这种方法。

```java
@ThreadSafe
public class BetterVector <E> extends Vector<E> {
    // When extending a serializable class, you should redefine serialVersionUID
    static final long serialVersionUID = -3963416950630760754L;

    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !contains(x);
        if (absent)
            add(x);
        return absent;
    }
}
```

“扩展”方法比直接将代码添加到类中更加脆弱，因为现在的同步策略实现被分布到多个单独维护的源代码文件中。如果底层的类改变了同步策略并选择了不同的锁来保护它的状态变量，那么子类会被破坏，因为在同步策略改变后它无法再使用正确的锁来控制对基类状态的并发访问。（在Vector的规范中定义了它的同步策略，因此BetterVector不存在这个问题。）

### 客户端加锁机制

对于Collections.synchronizedList封装的ArrayList，这两种方法在原始类中添加一个方法或者对类进行扩展都行不通，因为客户代码并不知道在同步封装器工厂方法中返回的List对象的类型。**第三种策略是扩展类的功能，但并不是扩展类本身，而是将扩展代码放入一个“辅助类”中**。

```java
@NotThreadSafe
class ListHelper <E> {
    public List<E> list = Collections.synchronizedList(new ArrayList<E>());
		......
    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !list.contains(x);
        if (absent)
            list.add(x);
        return absent;
    }
 
}
```

<!--上述代码不能实现线程安全的原因在于使用了不同的锁。-->必须使List在实现客户端加锁或外部加锁时使用同一个锁。客户端加锁是指，对于使用某个对象X的客户端代码，使用X本身用于保护保护其状态的锁来保护这段客户代码。要使用客户端加锁，你必须知道对象X使用的是哪一个锁。

在Vector和同步封装器类的文档中指出，他们通过Vector或同步封装器类的内置锁来支持客户端加锁。如下所示：

```java
@ThreadSafe
class ListHelper <E> {
    public List<E> list = Collections.synchronizedList(new ArrayList<E>());

    public boolean putIfAbsent(E x) {
        synchronized (list) {
            boolean absent = !list.contains(x);
            if (absent)
                list.add(x);
            return absent;
        }
    }
}
```

通过添加一个原子操作来扩展类是脆弱的，因为它将类的加锁代码分布到多个类中。然而，客户端加锁却更加脆弱，因为它将类C的加锁代码放到与C完全无关的其他类中。当在那些并不承诺遵循加锁策略的类上使用客户端加锁时，要特别小心。

**客户端加锁机制与扩展类机制有许多共同点，二者都是将派生类的行为与基类的实现耦合在一起。正如扩展会破坏实现的封装性，客户端加锁同样会破坏同步策略的封装性。**<!--在我看来，都不是好方法-->

### 组合

当为现有的类添加原子操作时，有一种更好的方法：组合。ImproveList通过将List对象的操作委托给底层的List实例来实现List的操作，同时还添加了一个原子的putIfAbsent方法。<!--与Collections.synchronizedList和其他容器封装器一样，ImproveList假设把某个链表对象传递给构造函数以后，客户代码不会再直接使用这个对象，而只能通过ImproveList来访问它。-->

```java
@ThreadSafe
public class ImprovedList<T> implements List<T> {
    private final List<T> list;

    /**
     * PRE: list argument is thread-safe.
     */
    public ImprovedList(List<T> list) { this.list = list; }

    public synchronized boolean putIfAbsent(T x) {
        boolean contains = list.contains(x);
        if (contains)
            list.add(x);
        return !contains;
    }

    // Plain vanilla delegation for List methods.
    // Mutative methods must be synchronized to ensure atomicity of putIfAbsent.

    public synchronized void clear() { list.clear(); }

   //... 按照类似的方式委托List的其他方法
}
```

ImproveList通过自身的内置锁增加了一层额外的加锁。它并不关心底层的List是否是线程安全的，即使List不是线程安全的或者修改了它的加锁实现，ImproveList也会提供一致的加锁机制来实现线程安全性。虽然额外的同步层可能导致轻微的性能损失，但与模拟另一个对象的加锁策略相比，ImproveList更为健壮。事实上，我们使用了Java监视器模式来封装现有的List，并且只要在类中拥有指向底层List的唯一外部引用，就能确保线程安全性。

## 4.5 将同步策略文档化

> 在文档中说明客户代码需要了解的线程安全性保证，以及代码维护人员需要了解的同步策略。

synchronized，volatile或者任何一个线程安全类都对应于某种同步策略，用于在并发访问时确保数据的完整性。这种策略的程序设计的要素之一，因此应该将其文档化。当然，在设计阶段时编写设计决策文档的最佳时间。这之后的几周或几个月后，一些设计细节会最逐渐变得模糊，因此一定要在忘记之前将它们记录下来。

在设计同步策略时需要考虑多个方面，例如，将哪些变量声明为volatile类型，哪些变量用锁来保护，哪些锁保护哪些变量，哪些变量必须是不可变的或者被封闭在线程中的，哪些操作必须是原子操作等。其中某些方面时严格的实现细节，应该将它们文档化以便于日后的维护。还有一些方面会影响类中加锁行为的外在表现，也应该将其视为规范的一部分写入文档。

我们认为“可能是线程安全”的类通常并不是线程安全的。

如果某个类没有明确地声明是线程安全的，那么就不要假设它是线程安全的，从而有效地避免类似于SimpleDateFormat的问题。

#### 解释含糊的文档

许多Java技术规范都没有说明接口的线程安全性，例如ServletContext，HttpSession或DataSource。

你只能去猜测。一个提高猜测准确性的方法是，从实现者（例如容器或数据库的供应商）的角度去解释规范，而不是从使用者的角度去解释。