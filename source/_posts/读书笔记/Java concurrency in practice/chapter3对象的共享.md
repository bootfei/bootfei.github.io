---
title: chapter3对象的共享
date: 2020-11-13 09:19:03
tags: [并发]
---

ref: 

1. java如何理解隐式地使this引用逸出
   https://segmentfault.com/q/1010000007900854
   https://www.cnblogs.com/viscu/p/9790755.html



Abstract：第二章开头指出，要编写正确的并发程序，关键在与：在访问共享的可变状态时，需要正确的管理。第二章通过**同步**，避免多个线程同时访问相同的变量；而本章介绍如何**共享**和**发布对象**，从而使多个线程可以同时安全的访问相同的变量。这两章合在一起构成了线程安全类的基础。



1. 可见性
   比如，由于"重排序"引起的读线程获取到**失效数据**，无法获取到主线程的最新写入的数据

2. 失效数据

   1. 比如一个类中只有get和set方法, 线程A调用set，同时线程B调用get，那么线程B可能看到失效数据，也可能看到最新数据。
   2. 拓展：类比Mysql中的read uncommitted隔离级别, 牺牲了准确性，但是提高了性能
   3. 如果对get和set都同步，那么此类就是线程安全

3. 加锁与可见性

   1. 锁可以保证原子性（或者说是互斥性），也可以保证可见性
   2. 锁保证可见性的原因在于：
      <img src="https://img-blog.csdnimg.cn/20190316230729391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5OTUxOTgz,size_16,color_FFFFFF,t_70" style="zoom:67%;" />

   为了确保所有线程都能看到共享变量的最新值，所以说所有执行的线程必须在同一个锁上同步

4. Volatile

   1. 稍弱的同步机制（作者应该是指的内存可见性这方面，而不是互斥性），将变量的更新操作通知到其他线程
   2. volatile作用在于可见性，不仅仅体现在使编译器不进行重排序，而且使得 ==> 当线程A写入volatile变量而线程B随后读取时，写入volatile变量之前对A可见的变量，在B读取了volatile以后，对B也是可见的，从内存可见性看，写入volatile=退出同步代码块（线程A），读volatile=进入同步代码块（线程B）
   3. 实际的应用场景
      1. 比如"数绵羊"，**检查某个状态标记**以判断是否退出循环
   4. 局限性：无法保证原子性
   5. 需要满足的条件，才能使用volatile：
      1. 在访问变量时，不需要加锁
      2. 变量不能与其他状态一起纳入不变性条件（因为volatile不保证原子性，而不变性条件需要原子性）
      3. 变量的写入操作依赖变量的当前值，或者确保只有一个线程（因为volatile不保证原子性，而依赖当前值，属于原子性）

5. 发布和逸出

   1. 概念：

      1. 发布：使对象能够在当前作用域之外的代码中使用（我自认为这个定义还不能对实际的分析有帮助，应该这样说：A发布了对象B，给对象或者线程C1orC2.....Cn使用，所以对象或者线程C1orC2在A的作用域之外使用对象B）
      2. 逸出：当某个不应该被发布的对象被发布时

   2. 场景：发布常见的场景，

      1. 将对象的引用存储到公共静态域，任何类和线程都能看到这个域
         比如，initailize实例化了knownSecrets，这个对象发布了knownSecrets

         ```java
         public static Set<Secret> knownSecrets;
         public void initialize() {
             knownSecrets = new HashSet<Secret>();
         }
         ```

      2. 发布一个对象间接地发布其他对象

         ```java
         //内部可变的数据逸出（不要这样做）
         class UnsafeStates {
             private String[] states = new String[] {
                 "AK", "AL" ...
             };
             public String[] getStates() { return states; }
         }
         ```

         比如，以这种方式发布states会出问题，这样会允许内部可变的数据逸出，请不要这样做。因为任何一个调用者都能修改它的内容。数组states已经逸出了它所属的范围，这个本就是私有的数据，事实上已经变成公有的了。
         ==>**如果一个已经发布的对象能够通过非私有的变量引用和方法调用到达其他的对象，那么这些对象也都会发布，会不安全。**

      3. 发布一个内部的类实例

         ```java
         public class ThisEscape {
             public ThisEscape(EventSource source) {
                 source.registerListener(new EventListener() {
                     public void onEvent(Event e) {
                         doSomething(e); //其实就是this.doSomething(e);
                     }
                 });
             }
         
             void doSomething(Event e) {
             }
         
         
             interface EventSource {
                 void registerListener(EventListener e);
             }
         
             interface EventListener {
                 void onEvent(Event e);
             }
         
             interface Event {
             }
         }
         ```

         ThisEscape发布EventListener时，也隐含地发布了ThisEscape实例本身，因为在这个内部类的实例中包含了对ThisEscape实例的隐含引用！！！
         *比如造成的后果，在多线程环境下，会出现这样一种情况：*
         *线程 A 和线程 B 同时访问 ThisEscape 构造方法，这时线程 A 访问构造方法还为完成(可以理解为 ThisEscape 为初始化完全)，此时由于 this 逸出，导致 this 在 A 和 B 中都具有可见性，线程 B 就可以通过 this 访问 doSomething(e) 方法，导致修改 ThisEscape 的属性。也就是在 ThisEscape 还为初始化完成，就被其他线程读取，导致出现一些奇怪的现象。*
         *这也就是 this 逸出。*

         ====> 得出结论：不要在构造过程中，使this引用逸出，具体来说，只有当构造函数返回时，this引用才应该从线程中逸出。解决方案如下:

         ​		如果想在**构造函数**中注册一个**事件监听器**或**启动线程**，那么可以使用私有的构造函数（保证构造函数必须先执行完毕）+公共的工厂方法

         ```java
         public class SafeListener {
             private final EventListener listener;
             private ThisEscape(EventSource source) {
                 listener = new EventListener(
                     public void onEvent(Event e) {
                         doSomething(e);
                     }
                 );
             }
             
             public static SafeLisener newInstance(EventSource source) {
                 SafeLisener safe = new SafeLisener();
                 soure.registerListener(safe.listener);
             }
             
         }
         ```

         

6. 线程封闭:

   1. 定义: 如果数据不共享，那么就不需要不同步，就是线程安全的，所以仅在单个线程内访问数据，称为线程封闭技术。

   2. 场景：

      1. Swing: 将所有组件和数据模型对象封闭到Swing的事件分发线程，除了事件线程以外的线程，都不能访问这些对象。
      2. JDBC的Connection对象: 在典型的服务器应用中，线程从连接池中获取一个Connection对象，处理完请求后，在将Connection对象返回给连接池。由于大多数请求，比如Servlet,都是由单个线程采用同步方式来处理，并且在Connection对象返回给连接池之前，连接池不会将它分配给其他线程。所以，Connection对象隐式地封闭在线程中。

   3. 技术实现:
      正如Java并没有强制规定某个变量必须由锁来保护，同样Java也无法强制对象必须在某个线程内，所以由程序负责确保封闭在线程内的对象会不会逸出，而Java只提供一些机制，比如局部变量和ThreadLocal类。

      1. Ad-hoc线程封闭: 

         1. 定义：维护线程封闭性的职责完全由**程序实现**来承担。
         2. 实际上，没有一个语言，能将对象封闭到一个线程上。所以，此方法还不如把程序设计为单线程，如Swing等Gui框架。

      2. 栈封闭:

         1. 定义: 是线程封闭的一种特例。只能通过局部变量才能访问对象，因为局部变量的固有属性就是封闭在执行线程中的，他们位于运行数据区的栈中（线程独享）。

      3. Threadlocal类

         1. 定义：这个类使线程中的某个值与保存值的对象关联起来。ThreadLocal提供了get与Set等访问接口和方法，这些方法为每个使用该变量的线程都存有一份独立的副本，所以，get（）会返回当前执行线程在调用set（）时设置的最新值

         2. 场景

            1. JDBC的连接对象不一定是线程安全的，而且希望避免调用每个方法时都传递一个Connection对象，因此，可以将JDBC的连接对象保存到ThreadLocal对象中。

               ```java
               public class ThreadLocalTemplate {
                   private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {
                       public Connection initialValue() {
                           return DriverManager.getConnection("DB_URL");  //JDBC的Connection不一定是线程安全的，封装到ThreadLocal中保证每个线程都会拥有自己的JDBC连接对象副本
                       }
                   };
                  
                   public static Connection getConnection() {
                       return connectionHolder.get();
                   }
               }
               ```

               

            2. 当某个频繁执行的操作需要一个临时对象，而同时希望避免在每次执行时都重新分配该临时对象。

               ```java
               Integer.toString();//JDK1.5使用ThreadLocal对象来保存12字节大小的缓存区
               ```

               

            3. 如果将单线程的应用程序移植到多线程环境中，只需要把全局变量转化为ThreadLocal对象（如果全局变量的语义允许），就可以维持线程安全性。
               注意：如果全局变量是应用程序用来作为缓存的，那此方法就是无意义的。因为缓存本来就是希望一个线程的计算结果可以被其他线程共享，而不是独享。

               

            4. J2EE的事物上下文(Transaction Context),保存在执行线程的ThreadLocal对象中。

         3. 原理：
            当执行线程首次调用ThreadLocal.get()时，就调用initialValue来获取初始值。从概念上看，ThreadLocal<T> 视为Map<Thread,T>对象，其中保存了特定于该线程的值，但是ThreadLocal的实现并非如此。这些特定于线程的值保存在Thread对象中，当线程终止时，这些值也被gc了。

7. 不变性：

   1. 定义：满足同步的另一种方法就是使用不可变的对象（某个对象在被创建后就不可被修改）
   2. 当满足以下条件时，对象才是不可变的：
      1. 对象创建以后其状态都不能修改
      2. 对象的所有域都是final类型
      3. 对象是正确创建的（在对象创建期间，this引用没有逸出）
   3. 技术实现：









