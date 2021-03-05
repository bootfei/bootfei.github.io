---
title: IO模型与多路复用
date: 2021-02-20 11:32:56
tags: [redis,IO模型]
---

# User space/Kernel space

学习 Linux 时，经常可以看到两个词:User space(用户空间)和 Kernel space(内核空间)。

虚拟内存被操作系统划分成两块:内核空间和用户空间，内核空间是内核代码运行的地方，用户空间是用户程序代码运行的地方。当进程运行在内核空间时就处于内核态，当进程运行在用户空间时就处于用户态。

查看 CPU 时间在 User space 与 Kernel Space 之间的分配情况，可以使用top命令。它的第三行输出就是 CPU 时间分配统计。

- 第一项24.8 us(user 的缩写)就是 CPU 消耗在 User space 的时间百分比，第二项0.5 sy(system 的 缩写)是消耗在 Kernel space 的时间百分比。
- 随便也说一下其他 6 个指标的含义。
- ni:niceness 的缩写，CPU 消耗在 nice 进程(低优先级)的时间百分比
- id:idle 的缩写，CPU 消耗在闲置进程的时间百分比，这个值越低，表示 CPU 越忙
- wa:wait 的缩写，CPU 等待外部 I/O 的时间百分比，这段时间 CPU 不能干其他事，但是也没有执行运算，这个 值太高就说明外部设备有问题
- hi:hardware interrupt 的缩写，CPU 响应硬件中断请求的时间百分比 si:software interrupt 的缩写，CPU 响应软件中断请求的时间百分比
- st:stole time 的缩写，该项指标只对虚拟机有效，表示分配给当前虚拟机的 CPU 时间之中，被同一台物理机上 的其他虚拟机偷走的时间百分比

# PIO与DMA

有必要简单地说说慢速I/O设备和内存之间的数据传输方式。

- PIO
  我们拿磁盘来说，很早以前，磁盘和内存之间的数据传输是需要CPU控制的，也就是说如果我们读取磁盘文件到内存中，数据要经过CPU存储转发，这种方式称为PIO。显然这种方式非常不合理，需要占用大量的CPU时间来读取文件，造成文件访问时系统几乎停止响应。
- DMA
  后来，DMA（直接内存访问，Direct Memory Access）取代了PIO，它可以不经过CPU而直接进行磁盘和内存的数据交换。在DMA模式下，CPU只需要向DMA控制器下达指令，让DMA控制器来处理数据的传送即可，DMA控制器通过系统总线来传输数据，传送完毕再通知CPU，这样就在很大程度上降低了CPU占有率，大大节省了系统资源，而它的传输速度与PIO的差异其实并不十分明显，因为这主要取决于慢速设备的速度。

可以肯定的是，PIO模式的计算机我们现在已经很少见到了。

# 缓存IO与直接IO

- 缓存IO:数据从磁盘先通过DMA copy到内核空间，再从内核空间通过cpu copy到用户空间 
- 直接IO:数据从磁盘通过DMA copy到用户空间

<img src="https://yqfile.alicdn.com/img_9ab2d009b5aa815127aeed8847a7f25d.jpeg" alt="img" style="zoom:67%;" />

## 缓存IO

<font color="red">缓存I/O又被称作标准I/O，大多数文件系统的默认I/O操作都是缓存I/O。</font>在Linux的缓存I/O机制中，数据先从磁盘复制到[内核空间缓冲区]()，然后从[内核空间缓冲区]()复制到[应用程序的地址空间]()。

- 读操作: 操作系统检查[内核缓冲区]()有没有需要的数据，如果已经缓存了，那么就直接从缓存中返回;否则从磁盘中读取，然后缓存在操作系统的缓存中。
- 写操作：将用户空间的数据复制到内核空间的缓存中。
  - 对于用户来说，写操作已经完成，至于内核空间何时将缓存写入磁盘，有操作系统决定。除非用户显示调用fsync命令

优缺点

- 优点：
  - 一定程度上分离了用户空间和系统空间，保障了系统的安全运行；
  - 减少了与磁盘IO的次数，提高了性能

- 缺点：
  - 在缓存 I/O 机制中，DMA 方式可以将数据直接从磁盘读到页缓存中，或者将数据从页缓存直接写回到磁盘上，而不能直接在应用程序地址空间和磁盘之间进行数据传输，这样数据在传输过程中需要在程序地址空间（用户空间）和缓存（内核空间）进行多次的数据复制操作，这些操作造成很大的CPU和内存消耗。

## 直接IO(`绕过内核缓冲区,自己管理I/O缓存区`)

<font color="red">直接IO就是应用程序直接访问磁盘数据，而不经过内核缓冲 区，也就是绕过内核缓冲区,自己管理I/O缓存区，这样做的目 的是减少一次从内核缓冲区到用户程序缓存的数据复制。</font>

引入[内核缓冲区]()的目的在于提高磁盘文件的访问性能，因为当进程需要读取磁盘文件时，如果文件内容已经在[内缓
冲区]()中，那么就不需要再次访问磁盘;而当进程需要向文件中写入数据时，实际上只是写到了[内核缓冲区]()便告诉进程已经写成功，而真正写入磁盘是通过一定的策略进行延迟的。

然而，对于一些较复杂的应用(MySQL)，它们为了充分提高性能，会绕过[内核缓冲区]()，由程序自己管理并且实现IO缓存，包括缓存机制和写延迟机制，以支持独特的查询机制，比如Mysql通过合理的策略来提高查询缓存命中率。另一方面，绕过[内核缓冲区]()也可以减少系统内存的开销，因为内核缓冲区本身就在使用系统内存。

<img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8yMzEyNDQ4Ni04YTE3MjI0ZDYyNDE2MDUx?x-oss-process=image/format,png" alt="image" style="zoom: 33%;" />

优缺点：

- 优点：
  - 应用程序直接访问磁盘数据，不经过操作系统[内核缓冲区]()，这样做的目的是减少一次[内核缓存]()与[程序缓存]()的复制。这种方式通常是在对数据的缓存管理由应用程序实现的数据库管理系统中。
- 缺点:
  - 如果被访问的数据不在程序缓存中，那么会从磁盘直接读取数据，非常低效的操作。

Linux提供了对这种需求的支持，即在open()系统调用中增加参数选项O_DIRECT，用它打开的文件便可以直接访问磁盘文件。

# IO访问方式

## 磁盘IO

<img src="https://yqfile.alicdn.com/img_41dd13543f263358e3ac143b5dcac23e.jpeg" alt="图片描述" style="zoom:75%;" />

当应用程序调用read接口时，操作系统检查在内核的高速缓存有没有需要的数据，如果已经缓存了，那么就直接从 缓存中返回，如果没有，则从磁盘中读取，然后缓存在操作系统的缓存中。

应用程序调用write接口时，将数据从用户地址空间复制到内核地址空间的缓存中，这时对用户程序来说，写操作已 经完成，至于什么时候再写到磁盘中，由操作系统决定，除非显示调用了sync同步命令。

## 网络IO`(I/Osendfile/零拷贝,kafka的特性`)

网络传输的第4层（传输层）TCP协议通过socket获取IO流完成的

### 普通的网络传输步骤

1）操作系统将数据从[磁盘hard drive]()复制到[内核缓存kernel buffer]()中
2）应用将数据从[内核缓存]()复制到[应用缓存user buffer]()中
3）应用将数据写回[内核的Socket缓存 (socket buffer)]()中
4）操作系统将数据从[Socket缓存]()区复制到[网卡缓存 (protocal engine)]()，然后将其通过网络发出

![图片描述](https://yqfile.alicdn.com/img_268ed262dadff71f51f42ad10ea46c9c.jpeg)



1、当调用read系统调用时,通过DMA（Direct Memory Access）将数据copy到内核模式
2、然后由CPU控制将内核模式数据copy到用户模式下的 buffer中
3、read调用完成后，write调用首先将用户模式下 buffer中的数据copy到内核模式下的socket buffer中
4、最后通过DMA copy将内核模式下的socket buffer中的数据copy到网卡设备中传送。

从上面的过程可以看出，数据白白从内核模式到用户模式走了一圈，浪费了两次copy，而这两次copy都是CPU copy，即占用CPU资源。



### sendFile

Linux2.4内核对sendfile做了改进，下图所示
![图片描述](https://yqfile.alicdn.com/img_8b7e24bd7f67281e97cd0cf6a32d513e.jpeg)
改进后的处理过程如下：
1、DMA copy将磁盘数据copy到kernel buffer中
2、向socket buffer中追加当前要发送的数据在kernel buffer中的位置和偏移量
3、DMA gather copy根据socket buffer中的位置和偏移量直接将kernel buffer中的数据copy到网卡上。
经过上述过程，数据只经过了2次copy就从磁盘传送出去了。（事实上这个Zero copy是针对内核来讲的，数据在内核模式下是Zero－copy的）。
当前许多高性能http server都引入了sendfile机制，如nginx，lighttpd等。

### FileChannel.transferTo(`Java中的零拷贝`)

Java NIO中FileChannel.transferTo(long position, long count, WriteableByteChannel target)方法将当前通道中的数据传送到目标通道target中，在支持Zero-Copy的linux系统中，transferTo()的实现依赖于 sendfile()调用。

![图片描述](https://yqfile.alicdn.com/img_80b6694edf8b127a0ab4f3a8763421e4.png)

传统方式对比零拷贝方式：

![图片描述](https://yqfile.alicdn.com/img_8cb2e11479b0cb46056998b6d32f1f3b.jpeg)

整个数据通路涉及4次数据复制和2个系统调用，如果使用sendfile则可以避免多次数据复制，操作系统可以**直接将数据从内核页缓存中复制到网卡缓存**，这样可以大大加快整个过程的速度。

大多数时候，我们都在向Web服务器请求静态文件，比如图片、样式表等，根据前面的介绍，我们知道在处理这些请求的过程中，磁盘文件的数据先要经过内核缓冲区，然后到达用户内存空间，因为是不需要任何处理的静态数据，所以它们又被送到网卡对应的内核缓冲区，接着再被送入网卡进行发送。

数据从内核出去，绕了一圈，又回到内核，没有任何变化，看起来真是浪费时间。在Linux 2.4的内核中，尝试性地引入了一个称为khttpd的内核级Web服务器程序，它只处理静态文件的请求。引入它的目的便在于内核希望请求的处理尽量在内核完成，减少内核态的切换以及用户态数据复制的开销。

同时，Linux通过系统调用将这种机制提供给了开发者，那就是sendfile()系统调用。它可以将磁盘文件的特定部分直接传送到代表客户端的socket描述符，加快了静态文件的请求速度，同时也减少了CPU和内存的开销。

在OpenBSD和NetBSD中没有提供对sendfile的支持。通过strace的跟踪看到了Apache在处理151字节的小文件时，使用了mmap()系统调用来实现内存映射，但是**在Apache处理较大文件的时候，内存映射会导致较大的内存开销，得不偿失**，所以Apache使用了sendfile64()来传送文件，sendfile64()是sendfile()的扩展实现，它在Linux 2.4之后的版本中提供。

这并不意味着sendfile在任何场景下都能发挥显著的作用。**对于请求较小的静态文件，sendfile发挥的作用便显得不那么重要**，通过压力测试，我们模拟100个并发用户请求151字节的静态文件，是否使用sendfile的吞吐率几乎是相同的，可见**在处理小文件请求时，发送数据的环节在整个过程中所占时间的比例相比于大文件请求时要小很多，所以对于这部分的优化效果自然不十分明显**。

## 网络IO和磁盘IO对比

- 磁盘IO主要的延时是由（以15000rpm硬盘为例）： 
  - 机械转动延时（机械磁盘的主要性能瓶颈，平均为2ms） + 寻址延时（2~3ms） + 块传输延时（一般4k每块，40m/s的传输速度，延时一般为0.1ms) 决定。（平均为5ms）

- 网络IO主要延时由： 
  - 服务器响应延时 + 带宽限制 + 网络延时 + 跳转路由延时 + 本地接收延时 决定。（一般为几十到几千毫秒，受环境干扰极大）

所以两者一般来说网络IO延时要大于磁盘IO的延时。



# Socket网络编程

第4层传输层TCP协议就是使用socket接收和发送IO流

## 客户端

```java
public class SocketClient {
  public static void main(String args[]) throws Exception {
      // 要连接的服务端IP地址和端口 
    	String host = "127.0.0.1"; int port = 55533;
      // 与服务端建立连接
      Socket socket = new Socket(host, port);
      // 建立连接后获得输出流
      OutputStream outputStream = socket.getOutputStream(); 
    	String message="你好 yiwangzhibujian"; 
    	socket.getOutputStream().write(message.getBytes("UTF-8"));
    	outputStream.close();
      socket.close();
} }
```



## 服务端

```java
public class SocketServer {
   public static void main(String args[]) throws Exception {
      // 监听指定的端口
      int port = 55533;
      ServerSocket server = new ServerSocket(port); // server将一直等待连接的到来 
     	System.out.println("server将一直等待连接的到来");
      //如果使用多线程，那就需要线程池，防止并发过高时创建过多线程耗尽资源 
      ExecutorService threadPool = Executors.newFixedThreadPool(100);
      while (true) {
      Socket socket = server.accept();
      Runnable runnable=()->{
          try {
            // 建立好连接后，从socket中获取输入流，并建立缓冲区进行读取 
            InputStream inputStream = socket.getInputStream(); 
            byte[] bytes = new byte[1024];
            int len;
            StringBuilder sb = new StringBuilder();
            while ((len = inputStream.read(bytes)) != -1) {
  // 注意指定编码格式，发送方和接收方一定要统一，建议使用UTF-8
              sb.append(new String(bytes, 0, len, "UTF-8"));
            }
            System.out.println("get message from client: " + sb);
            inputStream.close();
            socket.close();
          } catch (Exception e) {
            e.printStackTrace();
					} 
      };
      threadPool.submit(runnable);
    }
	} 
}
```



# 处理web请求的两种体系结构

在web服务中，处理web请求通常有两种体系结构，分别为：**thread-based architecture（基于线程的架构）、event-driven architecture（事件驱动模型）**

## thread-based architecture（基于线程的架构）

thread-based architecture（基于线程的架构），通俗的说就是：多线程并发模式，一个连接一个线程，服务器每当收到客户端的一个请求， 便开启一个独立的线程来处理。

<img src="https://pic1.zhimg.com/80/v2-288b2a61dbfcf488eefd4a6ab9ad08dc_1440w.jpg" alt="img" style="zoom:33%;" />

这种模式一定程度上极大地提高了服务器的吞吐量，由于在不同线程中，之前的请求在read阻塞以后，不会影响到后续的请求。**但是**，仅适用于于并发量不大的场景，因为：

- 线程需要占用一定的内存资源
- 创建和销毁线程也需一定的代价
- 操作系统在切换线程也需要一定的开销
- 线程处理I/O，在等待输入或输出的这段时间处于空闲的状态，同样也会造成cpu资源的浪费
- **如果连接数太高，系统将无法承受**

## event-driven architecture（事件驱动模型）

事件驱动体系结构是目前比较广泛使用的一种。这种方式会定义一系列的事件处理器来响应事件的发生，并且将**服务端接受连接**与**对事件的处理**分离。其中，**事件是一种状态的改变**。比如，tcp中socket的new incoming connection、ready for read、ready for write。

### Reactor模式

#### 简介

<font color="red">反应器设计模式(Reactor pattern)是一种为处理并发服务请求，并将请求提交到 一个或者多个服务处理程序的事件设计模式。当客户端请求抵达后，服务处理程序 使用多路分配策略，由一个非阻塞的线程来接收所有的请求，然后派发这些请求至 相关的工作线程进行处理。</font>

从这个描述中，我们知道Reactor模式**首先是事件驱动的，有一个或多个并发输入源，有一个Service Handler，有多个Request Handlers**；[Service Handler]()会对输入的[请求（Event）]()进行[多路复用]()，并同步地将它们分发给相应的[Request Handler]()。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191223084249141.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM0MTI3NzI=,size_16,color_FFFFFF,t_70)

![img](https://pic3.zhimg.com/80/v2-eae3b6613735ad11b30333825dae8b16_1440w.jpg)

**Reactor模式主要包含下面几部分内容:**

- [初始事件分发器(Initialization Dispatcher)]()：用于管理[Event Handler]()，定义注册、移除EventHandler等。它还作为Reactor模式的入口调用Synchronous Event Demultiplexer的select方法以阻塞等待事件返回，当阻塞等待返回时，根据事件发生的Handle将其分发给对应的Event Handler处理，即回调[Event Handler]()中的handle_event()方法
-  [同步（多路）事件分离器(Synchronous Event Demultiplexer)]()：无限循环等待新事件的到来，一旦发现有新的事件到来，就会通知[初始事件分发器]()去调取特定的[事件处理器]()。阻塞等待一系列的Handle中的事件到来，如果阻塞等待返回，即表示在返回的Handle中可以不阻塞的执行返回的事件类型。这个模块一般使用操作系统的select来实现。在Java NIO中用Selector来封装，当Selector.select()返回时，可以调用Selector的selectedKeys()方法获取Set
- [系统处理程序(Handle)]()：操作系统中的句柄，是对资源在操作系统层面上的一种抽象，它可以是打开的文件、一个连接(Socket)、Timer等。由于Reactor模式一般使用在网络编程中，因而这里一般指[Socket Handle]()，即一个网络连接（Connection，在Java NIO中的Channel）。这个Channel注册到Synchronous Event Demultiplexer中，以监听Handle中发生的事件，对ServerSocketChannnel可以是CONNECT事件，对SocketChannel可以是READ、WRITE、CLOSE事件等。
- [事件处理器(Event Handler)]()： 定义事件处理方法，以供Initialization Dispatcher回调使用。
- [Concrete Event Handler]()：事件EventHandler接口，实现特定事件处理逻辑。

**Reactor 类结构中包含有的主要角色：**

1. Handle：标示文件描述符
2. Event Demultiplexer：对操作系统内核实现I/O复用接口的封装等待发生事件发生
3. Event Handler：事件处理接口
4. Event Handler A/B：实现应用程序所提供的特定事件处理逻辑
5. Reactor：反应器定义一个接口，注册和删除关注的事件句柄、运行事件处理循环、等待就绪事件触发，分发事件到注册的回调函数。

[对于Reactor模式，可以将其看做由两部分组成，一部分是由Boss组成，另一部分是由worker组成]()。Boss就像老板一样，主要是拉活儿、谈项目，一旦Boss接到活儿了，就下发给下面的work去处理。

#### 业务流程时序图

<img src="https://img2020.cnblogs.com/blog/627770/202007/627770-20200712133918539-241195601.png" alt="Image result for Reactor 时序图" style="zoom: 50%;" />

1. 应用启动，将关注的事件handle注册到Reactor中;
2. 调用Reactor，进入无限事件循环，等待注册的事件到来;
3. 事件到来，select返回，Reactor将事件分发到之前注册的回调函数中处理;

#### 为什么使用Reactor模式

- 多线程模式：

```
为每个单独到来的请求，专门启动一条线程，这样的话造成系统的开销很大，并且在单核的机上，多线程并不能提高系
统的性能，除非在有一些阻塞的情况发生。否则线程切换的开销会使处理的速度变慢。
```

- Reactor模式：

```
服务器端启动一条单线程，用于轮询IO操作是否就绪，当有就绪的才进行相应的读写操作，这样的话就减少了服务器产 生大量的线程，也不会出现线程之间的切换产生的性能消耗。(目前JAVA的NIO就采用的此种模式，这里引申出一个问 题:在多核情况下NIO的扩展问题)
```

以上两种处理方式都是基于同步的，多线程的处理是我们传统模式下对高并发的处 理方式，Reactor模式的处理是现今面对高并发的主流方式。				 		



#### Reactor模式-单线程模式

Java中的NIO模式的Selector网络通讯，其实就是一个简单的Reactor模型。可以说是单线程的Reactor模式

![img](https://pic4.zhimg.com/80/v2-5f97abecc66698d6b1ce3034267e1fff_1440w.jpg)

Reactor的单线程模式的单线程主要是针对于I/O操作而言，也就是所以的I/O的accept()、read()、write()以及connect()操作都在一个线程上完成的。

但在目前的单线程Reactor模式中，不仅I/O操作在该Reactor线程上，连非I/O的业务操作也在该线程上进行处理了，这可能会大大延迟I/O请求的响应。所以我们应该将非I/O的业务逻辑操作从Reactor线程上卸载，以此来加速Reactor线程对I/O请求的响应。

#### Reactor模式-工作者线程池模式

与单线程模式不同的是，添加了一个**工作者线程池**，并将非I/O操作从Reactor线程中移出转交给工作者线程池（Thread Pool）来执行。这样能够提高Reactor线程的I/O响应，不至于因为一些耗时的业务逻辑而延迟对后面I/O请求的处理。

![img](https://pic3.zhimg.com/80/v2-2ffa44b686eea3ce55c7489fd67d1c1e_1440w.jpg)



在工作者线程池模式中，虽然非I/O操作交给了线程池来处理，但是**所有的I/O操作依然由Reactor单线程执行**，在高负载、高并发或大数据量的应用场景，依然较容易成为瓶颈。所以，对于Reactor的优化，又产生出下面的多线程模式。

#### Reactor模式-多线程模式

对于多个CPU的机器，为充分利用系统资源，将Reactor拆分为两部分：mainReactor和subReactor

![img](https://pic4.zhimg.com/80/v2-14b10c1dd4c45a1fe3fd92f91fffe2e3_1440w.jpg)

**mainReactor**负责监听server socket，用来处理网络新连接的建立，将建立的socket Channel指定注册给subReactor，通常**一个线程**就可以处理 ；

**subReactor**维护自己的selector, 基于mainReactor 注册的socketChannel多路分离I/O读写事件，读写网络数据，通常使用**多线程**；

对非I/O的操作，依然转交给工作者线程池（Thread Pool）执行。

此种模型中，每个模块的工作更加专一，耦合度更低，性能和稳定性也大量的提升，支持的可并发客户端数量可达到上百万级别。关于此种模型的应用，目前有很多优秀的框架已经在应用了，比如mina和netty 等。Reactor模式-多线程模式下去掉工作者线程池（Thread Pool），则是Netty中NIO的默认模式。

- mainReactor对应Netty中配置的BossGroup线程组，主要负责接受客户端连接的建立。一般只暴露一个服务端口，BossGroup线程组一般一个线程工作即可
- subReactor对应Netty中配置的WorkerGroup线程组，BossGroup线程组接受并建立完客户端的连接后，将网络socket转交给WorkerGroup线程组，然后在WorkerGroup线程组内选择一个线程，进行I/O的处理。WorkerGroup线程组主要处理I/O，一般设置`2*CPU核数`个线程

### Proactor模式



# 4种IO模型

在理解关于同步和阻塞的概念前，需要知道

```
I/0 操作 主要分成两部分
① 数据准备，将数据加载到内核缓存（数据加载到操作系统）
② 将内核缓存中的数据加载到用户缓存（从操作系统复制到应用中）
```

**同步和异步的概念描述的是用户线程与内核的交互方式**

**阻塞和非阻塞的概念描述的是用户线程调用内核IO操作的方式**

## 争议：**异步就是异步**

**![img](https://images2018.cnblogs.com/blog/874126/201808/874126-20180815160154943-682702591.png)**

![img](https://images2018.cnblogs.com/blog/874126/201808/874126-20180815160307369-420684321.png)

【**同步/异步】和【**阻塞/非阻塞】的关注点是存在区别的：

```
【同步/异步】表示是两个事件交互的是否有序依赖关系
同步：针对执行结果，A事件必须知道B事件的结果M后才执行得到结果。
异步：针对执行结果，执行A事件和执行B事件没有关系。

阻塞/非阻塞表示执行过程出现的状态
阻塞：针对执行者来说，执行A事件，执行过程因为条件未满足，执行状态变成等待状态。
非阻塞：针对执行者来说，就是事件A执行遇到未满足条件，执行另外独立的C事件。

总结：两者之间是没有关系的
【同步/异步】
   概念上是：事件A，B的结果之间的是否存在依赖关系；
   影响上是：保证依赖数据的正确性
【阻塞/非阻塞】
   概念上是：自身执行状态。
   影响上是：阻塞导致资源浪费。

特别注意：异步只有异步，同步才有阻塞和非阻塞的说法！

例子：
总整体看：传统的请求，是同步的（也是阻塞的），请求响应是有序的(请求响应之间也是等待的）；AJAX是异步请求（也是非阻塞的）。
同步不等于阻塞：
单个看：AJAX从客户端执行单个请求看数据是同步，但是执行是非阻塞，在未收到响应继续执行其他请求。
```



服务器端编程经常需要构造高性能的IO模型，常见的IO模型有四种: 

(1[)同步阻塞IO(Blocking IO)]():即传统的IO模型。Tomcat和Apache

(2)[同步非阻塞IO(Non-blocking IO)]():默认创建的socket都是阻塞的，非阻塞IO要求socket被设置为 NONBLOCK。注意这里所说的NIO并非Java的NIO(New IO)库。

(3)[IO多路复用(IO Multiplexing)]():即经典的Reactor设计模式，有时也称为异步阻塞IO，Java中的 Selector和Linux中的epoll都是这种模型。

(4)异步IO(Asynchronous IO):即经典的Proactor模式，也称为异步非阻塞IO



## 同步阻塞IO(Blocking IO)

<img src="https://img-blog.csdnimg.cn/20200806153255444.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MjYyMjY2,size_16,color_FFFFFF,t_70" alt="img" style="zoom:50%;" />



<img src="https://images2018.cnblogs.com/blog/874126/201808/874126-20180815215329688-621626362.png" alt="img" style="zoom:50%;" />

- [用户线程(上上图中的处理线程)]()通过[系统调用read发起IO读操作(上上图中的read)]()，由用户空间转到内核空间。内核等到数据包到达后，然后将 接收的数据拷贝到用户空间，完成read操作。
- 即用户需要等待read将socket中的数据读取到buffer后，才继续处理接收的数据。整个IO请求的过程中，用户线程 是被阻塞的，这导致用户在发起IO请求时，不能做任何事情，对CPU的资源利用率不够。

```java
{
    read(socket, buffer); //阻塞
    process(buffer);
}
```

**模型特点 ：**

- 采用阻塞IO模式获取输入的数据
- 每个连接都需要独立的线程完成数据的输入，业务处理，数据返回

**问题分析：**

- 当并发数很大，就会创建大量的线程，占用很大系统资源
- 连接创建后，如果当前线程暂时没有数据可读，该线程会阻塞在read 操作，造成线程资源浪费

## 同步非阻塞IO

<img src="https://images0.cnblogs.com/blog/405877/201411/142332004602984.png" alt="img" style="zoom:50%;" />

同步非阻塞IO是在同步阻塞IO的基础上，将socket设置为NONBLOCK。这样做用户线程可以在发起IO请求后可以立即 返回。

由于socket是非阻塞的方式，因此用户线程发起IO请求时立即返回。但并未读取到任何数据，用户线程 需要不断地发起IO请求，直到数据到达后，才真正读取到数据，继续执行。

```java
{
	while(read(socket, buffer) != SUCCESS);
	process(buffer);
}
```

可以看到，用户线程需要不停地发送系统调用以获取这个 I/O 操作的最新状态，这个过程称为**轮询（poll）**。

虽然用户线程不再被阻塞了，但用户线程需要不断地进行轮询，轮询过程会消耗额外的 CPU 资源。因此 CPU 的有效利用率同样不高。

## IO多路复用

<img src="https://img-blog.csdnimg.cn/20200806154511111.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4MjYyMjY2,size_16,color_FFFFFF,t_70" alt="img" style="zoom: 33%;" />

### 针对传统阻塞 I/O 模型的 2 个缺点的改进：（IO多路复用 + 线程池）

1、每个连接创建一个线程：（线程池）

- 基于线程池复用线程资源

2、每个连接对应一个阻塞对象：（IO多路复用）

- 多个连接共用一个阻塞对象，应用程序只需要在[一个阻塞对象(Service Handler)]()等待，无需阻塞等待所有连接。当某个连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理



### Select函数

IO多路复用模型是建立在内核提供的多路分离函数select基础之上的，使用select函数可以避免同步非阻塞IO模型中轮询等待的问题。

<img src="https://images0.cnblogs.com/blog/405877/201411/142332187256396.png" alt="img" style="zoom: 25%;" />



1. 用户首先将需要进行IO操作的socket添加到select中 <!--正如 new ServerSocket(port)，对某个port进行监控和IO操作-->
2. 然后阻塞等待select系统调用返回。
3. 当数据到达时，socket被激活，select函数返回。用户线程正式发起read请求，读取数据并继续执行。

> 从流程上来看，使用select函数进行IO请求和同步阻塞模型没有太大的区别，甚至还多了添加监视socket，以及调用select函数的额外操作，效率更差。但是，[使用select以后最大的优势是用户可以在一个线程内同时处理多个socket的IO请求]()。用户可以注册多个socket，然后不断地调用select读取被激活的socket，即可达到在<font color="red">**同一个线程内同时处理多个IO请求的目的**</font>。而在同步阻塞模型中，必须通过多线程的方式才能达到这个目的。

用户线程 <!--相对于OS中的内核，server确实是OS中的用户线程--> 使用select函数的伪代码描述为：

```java
{
    select(socket);
  	//其中while循环前将socket添加到select监视中，然后在while内一直调用select获取被激活的socket，一旦socket可读，便调用read函数将socket中的数据读取出来。
    while(1) {
        sockets = select();
        for(socket in sockets) {
            if(can_read(socket)) {
                read(socket, buffer);
                process(buffer);
            }
        }
    }
}
```

然而，使用select函数的优点并不仅限于此。虽然上述方式允许单线程内处理多个IO请求 <!--一个线程通过无线循环执行socket = select()来处理每个socket请求-->，但是每个IO请求的过程还是阻塞的（[在select函数上阻塞]()）<!--select()要等待监听的socket有数据传过来-->，平均时间甚至比同步阻塞IO模型还要长。如果用户线程只注册自己感兴趣的socket或者IO请求，然后去做自己的事情，等到数据到来时再进行处理，则可以提高CPU的利用率。<!--就是将这张图中的用户线程，换成一个专门的线程来处理阻塞事务-->

### Reactor设计模式

IO多路复用模型使用了Reactor设计模式实现了这一机制。

<img src="https://images0.cnblogs.com/blog/405877/201411/142332350853195.png" alt="img" style="zoom: 33%;" />

**图4 Reactor设计模式**

如图4所示，EventHandler抽象类表示IO事件处理器，它拥有IO文件句柄Handle（通过get_handle获取），以及对Handle的操作handle_event（读/写等）。继承于EventHandler的子类可以对事件处理器的行为进行定制。Reactor类用于管理EventHandler（注册、删除等），并使用handle_events实现事件循环，不断调用同步事件多路分离器（一般是内核）的多路分离函数select，只要某个文件句柄被激活（可读/写等），select就返回（阻塞），handle_events就会调用与文件句柄关联的事件处理器的handle_event进行相关操作。

<img src="https://images0.cnblogs.com/blog/405877/201411/142333254136604.png" alt="img" style="zoom:33%;" />

**图5 IO多路复用**

如图5所示，通过Reactor的方式，可以将用户线程轮询IO操作状态的工作统一交给handle_events事件循环进行处理。用户线程注册事件处理器之后可以继续执行做其他的工作（异步），而Reactor线程负责调用内核的select函数检查socket状态。当有socket被激活时，则通知相应的用户线程（或执行用户线程的回调函数[handle_event]()），执行handle_event进行数据读取、处理的工作。由于select函数是阻塞的，因此多路IO复用模型也被称为异步阻塞IO模型。[注意，这里的所说的阻塞是指select函数执行时线程被阻塞，而不是指socket]()。一般在使用IO多路复用模型时，socket都是设置为NONBLOCK的 <!--与同步非阻塞IO一样-->，不过这并不会产生影响，因为[用户线程]()发起[IO请求（read请求）]()时，数据已经到达了，用户线程一定不会被阻塞  <!--用户线程read请求时，Reactor线程已经通知用户线程socket可读了-->。

[用户线程]()使用IO多路复用模型的伪代码描述为：<!--用户线程有很多，process是具体的业务逻辑-->

```java
void UserEventHandler::handle_event() {
    if(can_read(socket)) {
        read(socket, buffer);
        process(buffer);
    }
}

{
		Reactor.register(new UserEventHandler(socket));
}
```

[用户线程]()需要重写EventHandler的handle_event函数进行读取数据、处理数据的工作，[用户线程]()只需要将自己的EventHandler注册到Reactor即可。Reactor中handle_events事件循环的伪代码大致如下。<!--其实和上述的select函数方法一样,Reactor只有一个线程-->

```java
Reactor::handle_events() {
    while(1) {
        sockets = select();
        for(socket in sockets) {
       		 get_event_handler(socket).handle_event();
        }
    }
}
```

事件循环不断地调用select获取被激活的socket，然后根据获取socket对应的EventHandler，执行器handle_event函数即可。

IO多路复用是最常使用的IO模型，但是其异步程度还不够“彻底”，因为它使用了会阻塞线程的select系统调用 <!--就是select()方法会阻塞Reactor线程-->。因此IO多路复用只能称为异步阻塞IO，而非真正的异步IO。



## 异步IO（不是重点）

“真正”的异步IO需要操作系统更强的支持。在IO多路复用模型中，事件循环将文件句柄的状态事件通知给用户线程，由用户线程自行读取数据、处理数据。而在异步IO模型中，当用户线程收到通知时，数据已经被内核读取完毕，并放在了用户线程指定的缓冲区内，内核在IO完成后通知用户线程直接使用即可。

异步IO模型使用了Proactor设计模式实现了这一机制。

![img](https://images0.cnblogs.com/blog/405877/201411/151608309061672.jpg)

图6 Proactor设计模式

如图6，Proactor模式和Reactor模式在结构上比较相似，不过在用户（Client）使用方式上差别较大。Reactor模式中，用户线程通过向Reactor对象注册感兴趣的事件监听，然后事件触发时调用事件处理函数。而Proactor模式中，用户线程将AsynchronousOperation（读/写等）、Proactor以及操作完成时的CompletionHandler注册到AsynchronousOperationProcessor。AsynchronousOperationProcessor使用Facade模式提供了一组异步操作API（读/写等）供用户使用，当用户线程调用异步API后，便继续执行自己的任务。AsynchronousOperationProcessor 会开启独立的内核线程执行异步操作，实现真正的异步。当异步IO操作完成时，AsynchronousOperationProcessor将用户线程与AsynchronousOperation一起注册的Proactor和CompletionHandler取出，然后将CompletionHandler与IO操作的结果数据一起转发给Proactor，Proactor负责回调每一个异步操作的事件完成处理函数handle_event。虽然Proactor模式中每个异步操作都可以绑定一个Proactor对象，但是一般在操作系统中，Proactor被实现为Singleton模式，以便于集中化分发操作完成事件。

<img src="https://images0.cnblogs.com/blog/405877/201411/142333511475767.png" alt="img" style="zoom:25%;" />

图7 异步IO

如图7所示，异步IO模型中，用户线程直接使用内核提供的异步IO API发起read请求，且发起后立即返回，继续执行用户线程代码。不过此时用户线程已经将调用的AsynchronousOperation和CompletionHandler注册到内核，然后操作系统开启独立的内核线程去处理IO操作。当read请求的数据到达时，由内核负责读取socket中的数据，并写入用户指定的缓冲区中。最后内核将read的数据和用户线程注册的CompletionHandler分发给内部Proactor，Proactor将IO完成的信息通知给用户线程（一般通过调用用户线程注册的完成事件处理函数），完成异步IO。

用户线程使用异步IO模型的伪代码描述为：

```
void UserCompletionHandler::handle_event(buffer) {
		process(buffer);
}

{
		aio_read(socket, new UserCompletionHandler);
}
```

用户需要重写CompletionHandler的handle_event函数进行处理数据的工作，参数buffer表示Proactor已经准备好的数据，用户线程直接调用内核提供的异步IO API，并将重写的CompletionHandler注册即可。

相比于IO多路复用模型，异步IO并不十分常用，不少高性能并发服务程序使用IO多路复用模型+多线程任务处理的架构基本可以满足需求。况且目前操作系统对异步IO的支持并非特别完善，更多的是采用IO多路复用模型模拟异步IO的方式（IO事件触发时不直接通知用户线程，而是将数据读写完毕后放到用户指定的缓冲区中）。Java7之后已经支持了异步IO，感兴趣的读者可以尝试使用。





# Redis使用多路复用

redis 是一个单线程却性能非常好的内存数据库， 主要用来作为缓存系统。redis采用网络IO多路复用技术来保证在多连接的时候， 系统的高吞吐量。

## 为什么 Redis 中要使用 I/O 多路复用这种技术呢？

首先，Redis 是跑在单线程中的，所有的操作都是按照顺序线性执行的，但是由于读写操作等待用户输入或输出都是阻塞的，所以 I/O 操作在一般情况下往往不能直接返回，这会导致某一文件的 I/O 阻塞导致整个进程无法对其它客户提供服务，而 I/O 多路复用就是为了解决这个问题而出现的。<!--redis单线程机制，天生只能使用IO多路复用-->

redis的io模型主要是基于epoll实现的，不过它也提供了 select和kqueue的实现，默认采用epoll。

epoll是众多i/o多路复用技术当中的一种，相比其他io多路复用技术(select, poll等等)，epoll有诸多优点：

- epoll 没有最大并发连接的限制，上限是最大可以打开文件的数目，这个数字一般远大于 2048, 一般来说这个数目和系统内存关系很大 ，具体数目可以 cat /proc/sys/fs/file-max 察看。
- 效率提升， Epoll 最大的优点就在于它只管你“活跃”的连接 ，而跟连接总数无关，因此在实际的网络环境中， Epoll 的效率就会远远高于 select 和 poll 。
- 内存拷贝， Epoll 在这点上使用了“共享内存 ”，这个内存拷贝也省略了。

## epoll与select/poll的区别

   select，poll，epoll都是IO多路复用的机制。I/O多路复用就通过一种机制，可以监视多个描述符，一旦某个描述符就绪，能够通知程序进行相应的操作。

   select的本质是采用32个整数的32位，即32*32= 1024来标识，fd值为1-1024。当fd的值超过1024限制时，就必须修改FD_SETSIZE的大小。这个时候就可以表示32*max值范围的fd。

   poll与select不同，通过一个pollfd数组向内核传递需要关注的事件，故没有描述符个数的限制，pollfd中的events字段和revents分别用于表示关注的事件和发生的事件，故pollfd数组只需要被初始化一次。

   epoll还是poll的一种优化，返回后不需要对所有的fd进行遍历，在内核中维持了fd的列表。select和poll是将这个内核列表维持在用户态，然后传递到内核中。与poll/select不同，epoll不再是一个单独的系统调用，而是由epoll_create/epoll_ctl/epoll_wait三个系统调用组成，后面将会看到这样做的好处。epoll在2.6以后的内核才支持。

**2.1 select/poll的几大缺点：**

- 每次调用select/poll，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
- 同时每次调用select/poll都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也很大
- 针对select支持的文件描述符数量太小了，默认是1024
- select返回的是含有整个句柄的数组，应用程序需要遍历整个数组才能发现哪些句柄发生了事件；
- select的触发方式是水平触发，应用程序如果没有完成对一个已经就绪的文件描述符进行IO操作，那么之后每次select调用还是会将这些文件描述符通知进程。

相比select模型，poll使用链表保存文件描述符，因此没有了监视文件数量的限制，但其他三个缺点依然存在。

## epoll IO多路复用模型实现机制

由于epoll的实现机制与select/poll机制完全不同，上面所说的 select的缺点在epoll上不复存在。

epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左右，设想一下如下场景：有100万个客户端同时与一个服务器进程保持着TCP连接。而每一时刻，通常只有几百上千个TCP连接是活跃的(事实上大部分场景都是这种情况)。如何实现这样的高并发？

在select/poll时代，服务器进程每次都把这100万个连接告诉操作系统(从用户态复制句柄数据结构到内核态)，让操作系统内核去查询这些套接字上是否有事件发生，轮询完后，再将句柄数据复制到用户态，让服务器应用程序轮询处理已发生的网络事件，这一过程资源消耗较大，因此，select/poll一般只能处理几千的并发连接。

如果没有I/O事件产生，我们的程序就会阻塞在select处。但是依然有个问题，我们从select那里仅仅知道了，有I/O事件发生了，但却并不知道是那几个流（可能有一个，多个，甚至全部），我们只能无差别轮询所有流，找出能读出数据，或者写入数据的流，对他们进行操作。

但是使用select，我们有O(n)的无差别轮询复杂度，同时处理的流越多，每一次无差别轮询时间就越长

epoll的设计和实现与select完全不同。epoll通过在Linux内核中申请一个简易的文件系统(文件系统一般用什么数据结构实现？B+树)。把原先的select/poll调用分成了3个部分：

- 调用epoll_create()建立一个epoll对象(在epoll文件系统中为这个句柄对象分配资源)
- 调用epoll_ctl向epoll对象中添加这100万个连接的套接字
- 调用epoll_wait收集发生的事件的连接

如此一来，要实现上面说是的场景，只需要在进程启动时建立一个epoll对象，然后在需要的时候向这个epoll对象中添加或者删除连接。同时，epoll_wait的效率也非常高，因为调用epoll_wait时，并没有一股脑的向操作系统复制这100万个连接的句柄数据，内核也不需要去遍历全部的连接。

## epoll IO底层实现

当某一进程调用epoll_create方法时，Linux内核会创建一个eventpoll结构体，这个结构体中有两个成员与epoll的使用方式密切相关。eventpoll结构体如下所示：

![][redis epoll底层实现eventpoll]

每一个epoll对象都有一个独立的eventpoll结构体，用于存放通过epoll_ctl方法向epoll对象中添加进来的事件。这些事件都会挂载在红黑树中，如此，重复添加的事件就可以通过红黑树而高效的识别出来(红黑树的插入时间效率是lgn，其中n为树的高度)。

而所有添加到epoll中的事件都会与设备(网卡)驱动程序建立回调关系，也就是说，当相应的事件发生时会调用这个回调方法。这个回调方法在内核中叫ep_poll_callback,它会将发生的事件添加到rdlist双链表中。

在epoll中，对于每一个事件，都会建立一个epitem结构体，如下所示：

![Image result for redis epoll底层实现 红黑树](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQdae_utkuyKOqbey3lfFOJz0CHH6V_-QNQ3w&usqp=CAU)

当调用epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可。如果rdlist不为空，则把发生的事件复制到用户态，同时将事件数量返回给用户。

优势：

**1）. 不用重复传递。**

我们调用epoll_wait时就相当于以往调用select/poll，但是这时却不用传递socket句柄给内核，因为内核已经在epoll_ctl中拿到了要监控的句柄列表。

 **2）. 在内核里，一切皆文件**

epoll向内核注册了一个文件系统，用于存储上述的被监控socket。当你调用epoll_create时，就会在这个虚拟的epoll文件系统里创建一个file结点。当然这个file不是普通文件，它只服务于epoll。

epoll在被内核初始化时（操作系统启动），同时会开辟出epoll自己的内核高速cache区，用于安置每一个我们想监控的socket，这些socket会以红黑树的形式保存在内核cache里，以支持快速的查找、插入、删除。这个内核高速cache区，就是建立连续的物理内存页，然后在之上建立slab层，简单的说，就是物理上分配好你想要的size的内存对象，每次使用时都是使用空闲的已分配好的对象。

 **3）. 极其高效的原因：**

这是由于我们在调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个红黑树用于存储以后epoll_ctl传来的socket外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。

当我们执行epoll_ctl时，除了把socket放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把它放到准备就绪list链表里。所以，当一个socket上有数据到了，内核在把网卡上的数据copy到内核中后就来把socket插入到准备就绪链表里了。从上面这句可以看出，epoll的基础就是回调！ 

一颗红黑树，一张准备就绪句柄链表，少量的内核cache，就帮我们解决了大并发下的socket处理问题。执行epoll_create时，创建了红黑树和就绪链表，执行epoll_ctl时，如果增加socket句柄，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪链表中插入数据。执行epoll_wait时立刻返回准备就绪链表里的数据即可。





# 附录

## redis epoll底层实现红黑树

[redis epoll底层实现eventpoll]:data:image/jpeg;base64,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