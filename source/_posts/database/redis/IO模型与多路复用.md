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

Reactor模式和Proactor模式都是是event-driven architecture（事件驱动模型）的实现方式

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

好比一个餐馆，

1. 有5桌客人，那就派5个服务员全程负责这桌客人的点菜、传单、结账等；有10桌客人，那就派10个服务员；....如果客人非常多，那就得多少服务员啊！！
2. 那么就使用线程池，总共有5个服务员。但是有的客人等不及，造成客人流水
3. 那么就使用Reactor模式，客人在点单的时候，服务员去负责其他客人，等客人点好菜了，叫服务员一声就行了。所以，只需要一个服务员负责所有客人的点菜就好了。

**并发系统常使用reactor模式代替常用的多线程的处理方式，节省系统的资源，提高 系统的吞吐量。**

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

流程与Reactor模式类似，区别在于proactor在IO ready事件触发后，完成IO操作再通知应用回调。虽然在linux平台还是基于epoll/select，但是内部实现了异步操作处理器(Asynchronous Operation Processor)以及异步事件分离器(Asynchronous Event Demultiplexer)将IO操作与应用回调隔离。经典应用例如boost asio异步IO库的结构和流程图如下：

![img](https://pic4.zhimg.com/80/v2-3ed3d63b31460c562e43dfd32d808e9b_1440w.jpg)

再直观一点，就是下面这幅图：



![img](https://pic1.zhimg.com/80/v2-ae0c50cb3b3480fc36b8614b8b77f528_1440w.jpg)



再再直观一点，其实就回到了五大模型-异步I/O模型的流程，就是下面这幅图：



![img](https://pic3.zhimg.com/80/v2-557eee325d2e29665930825618f7b212_1440w.jpg)

针对第二幅图在稍作解释：

Reactor模式中，用户线程通过向Reactor对象注册感兴趣的事件监听，然后事件触发时调用事件处理函数。而Proactor模式中，用户线程将AsynchronousOperation（读/写等）、Proactor以及操作完成时的CompletionHandler注册到AsynchronousOperationProcessor。

AsynchronousOperationProcessor使用Facade模式提供了一组异步操作API（读/写等）供用户使用，当用户线程调用异步API后，便继续执行自己的任务。AsynchronousOperationProcessor 会开启独立的内核线程执行异步操作，实现真正的异步。当异步IO操作完成时，AsynchronousOperationProcessor将用户线程与AsynchronousOperation一起注册的Proactor和CompletionHandler取出，然后将CompletionHandler与IO操作的结果数据一起转发给Proactor，Proactor负责回调每一个异步操作的事件完成处理函数handle_event。虽然Proactor模式中每个异步操作都可以绑定一个Proactor对象，但是一般在操作系统中，Proactor被实现为Singleton模式，以便于集中化分发操作完成事件。

### Reactor模式和Proactor模式的总结对比

#### 2.3.1 主动和被动

以主动写为例：

- Reactor将handler放到select()，等待可写就绪，然后调用write()写入数据；写完数据后再处理后续逻辑；
- Proactor调用aoi_write后立刻返回，由内核负责写操作，写完后调用相应的回调函数处理后续逻辑

**Reactor模式是一种被动的处理**，即有事件发生时被动处理。而**Proator模式则是主动发起异步调用**，然后循环检测完成事件。

#### 2.3.2 实现

Reactor实现了一个被动的事件分离和分发模型，服务等待请求事件的到来，再通过不受间断的同步处理事件，从而做出反应；

Proactor实现了一个主动的事件分离和分发模型；这种设计允许多个任务并发的执行，从而提高吞吐量。

所以涉及到文件I/O或耗时I/O可以使用Proactor模式，或使用多线程模拟实现异步I/O的方式。

#### 2.3.3 优点

Reactor实现相对简单，对于链接多，但耗时短的处理场景高效；

- 操作系统可以在多个事件源上等待，并且避免了线程切换的性能开销和编程复杂性；
- 事件的串行化对应用是透明的，可以顺序的同步执行而不需要加锁；
- 事务分离：将与应用无关的多路复用、分配机制和与应用相关的回调函数分离开来。

Proactor在**理论上**性能更高，能够处理耗时长的并发场景。为什么说在**理论上**？请自行搜索Netty 5.X版本废弃的原因。

#### 2.3.4 缺点

Reactor处理耗时长的操作会造成事件分发的阻塞，影响到后续事件的处理；

Proactor实现逻辑复杂；依赖操作系统对异步的支持，目前实现了纯异步操作的操作系统少，实现优秀的如windows IOCP，但由于其windows系统用于服务器的局限性，目前应用范围较小；而Unix/Linux系统对纯异步的支持有限，应用事件驱动的主流还是通过select/epoll来实现。

#### 2.3.5 适用场景

Reactor：同时接收多个服务请求，并且依次同步的处理它们的事件驱动程序；

Proactor：异步接收和同时处理多个服务请求的事件驱动程序。





