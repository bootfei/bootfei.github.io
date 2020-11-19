---
title: chapter3对象的共享
date: 2020-11-13 09:19:03
tags: [并发]
---

ref: 

1. java如何理解隐式地使this引用逸出
   https://segmentfault.com/q/1010000007900854



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









