---
title: chapter04-默认连接器
date: 2021-05-19 08:32:39
tags:
---

## 简介

第三章的连接器只是一个学习版，是为了介绍tomcat的默认连接器而写。第四章会深入讨论下tomcat的默认连接器（这里指的是tomcat4的默认连接器，现在该连接器已经不推荐使用，而是被Coyote取代）。

​     tomcat的连接器是一个独立的模块，可被插入到servlet容器中。目前已经有很多连接器的实现，包括Coyote，mod_jk，mod_jk2，mod_webapp等。tomcat的连接器需要满足以下要求：

​     （1）实现org.apache.catalina.Connector接口；

​     （2）负责创建实现了org.apache.catalina.Request接口的request对象；

​     （3）负责创建实现了org.apache.catalina.Response接口的response对象。

​     tomcat4的连接器与第三章实现的连接器类似，等待http请求，创建request和response对象，调用org.apache.catalina.Container的invoke方法将request对象和response对象传入container。在invoke方法中，container负责载入servlet类，调用其call方法，管理session，记录日志等工作

​     tomcat的默认连接器中有一些优化操作没有在chap3的连接器中实现。首先是提供了一个对象池，避免频繁创建一些创佳代价高昂的对象。其次，默认连接器中很多地方使用了字符数组而非字符串。

​     本章的程序是实现一个使用默认连接器的container。但，本章的重点不在于container，而是connector。另一个需要注意的是，默认的connector实现了HTTP1.1，也可以服务HTTP1.0和HTTP0.9的客户端。

​     本章以HTTP1.1的3个新特性开始，这对于理解默认connector的工作机理很重要。然后，要介绍org.apache.catalina.Connector接口



## HTTP1.1新特性

持久连接

块编码

状态码100的使用



## Connector接口

tomcat的connector必须实现org.apache.catalina.Connector接口。该接口有很多方法，最重要的是getContainer，setContainer，createRequest和createResponse。

setContainer方法用于将connector和container联系起来，getContainer则可以返回响应的container，createRequest和createResponse则分别负责创建request和response对象。

 org.apache.catalina.connector.http.HttpConnector类是Connector接口的一个实现，将在下一章讨论。响应的uml图如下所示

<img src="http://sishuok.com/forum/upload/2012/4/12/35840e616ec2b6f657d035945d0bd813__%E6%9C%AA%E5%91%BD%E5%90%8D.jpg" alt="img" style="zoom:75%;" />

> 注意，connector和container是一对一的关系，而connector和processor是一对多的关系。





## HttpConnector类

在第三章中，已经实现了一个与org.apache.catalina.connector.http.HttpConnector类似的简化版connector。它实现了org.apache.catalina.Connector接口，java.lang.Runnable接口（确保在自己的线程中运行）和org.apache.catalina.Lifecycle接口。Lifecycle接口用于维护每个实现了该接口的tomcat的组件的生命周期。

Lifecycle具体内容将在第六章介绍。实现了Lifecycle接口后，当创建一个HttpConnector实例后，就应该调用其initialize方法和start方法。在组件的整个生命周期内，这两个方法只应该被调用一次。下面要介绍一些与第三章不同的功能：创建ServerSocket，维护HttpProcessor池，提供Http请求服务。

### 创建ServerSocket

HttpConnector的initialize方法会调用一个私有方法open，返回一个java.net.ServerSocket实例，赋值给成员变量serverSocket。这里并没有直接调用ServerSocket的构造方法，而是用过open方法调用ServerSocket的一个工厂方法来实现。具体的实现方式可参考ServerSocketFactory类和DefaultServerSocketFactory类（都在org.apache.catalina.net包内）。

### 维护HttpProcessor对象池

在第三章的程序中，每次使用HttpProcessor时，都会创建一个实例。而在tomcat的默认connector中，使用了一个HttpProcessor的对象池， 其中的每个对象都在其自己的线程中使用。因此，connector可同时处理多个http请求。

HttpConnector维护了一个HttpProcessor的对象池，避免了频繁的创建HttpProcessor对象。该对象池使用java.io.Stack实现。

在HttpConnector中，创建的HttpProcessor数目由两个变量决定：minProcessors和maxProcessors。

```
protected int minProcessors = 5;

private int maxProcessors = 20;
```

默认情况下，minProcessors=5，maxProcessors=20，可通过其setter方法修改。

​     初始化的时候，HttpConnector会创建minProcessors个HttpProcessor对象。若不够用就继续创建，直到到达maxProcessors个。此时，若还不够，则后达到的http请求将被忽略。若是不希望对maxProcessors进行限制，可以将其置为负数。此外，变量curProcessors表示当前已有的HttpProcessor实例数目。

​     下面是start方法中初始化HttpProcessor对象的代码：

```java
while (curProcessors < minProcessors) {   
     if ((maxProcessors > 0) && (curProcessors >= maxProcessors))   
       break;   
     HttpProcessor processor = newProcessor();   
     recycle(processor);   
}  
```

其中newProcessor方法负责创建HttpProcessor实例，并将curProcessors加1。recycle方法将新创建的HttpProcessor对象入栈。

每个HttpProcessor对象负责解析请求行和请求头，填充request对象。因此，每个HttpProcessor对象都关联一个request对象和response对象。HttpProcessor的构造函数会调用HttpConnector的createRequest方法和createResponse方法。

### 提供Http请求服务

HttpConnector类的主要业务逻辑在其run方法中（例如第三章的程序中那样）。run方法中维持一个循环体，该循环体内，服务器等待http请求，直到HttpConnector对象回收。

```java
while (!stopped) {   
     Socket socket = null;   
     try {   
       socket = serverSocket.accept();   
     ...  
```

对于每个http请求，通过调用其私有方法createProcessor获得一个HttpProcessor对象。这里，实际上是从HttpProcessor的对象池中拿一个对象。

注意，若是此时对象池中已经没有空闲的HttpProcessor实例可用，则createProcessor返回null。此时，服务器会直接关闭该连接，忽略该请求。如代码所示：

```java
if (processor == null) {   
       try {   
         log(sm.getString("httpConnector.noProcessor"));   
         socket.close();   
       }   
       ...   
       continue;    
```

若是createProcessor方法返回不为空，则调用该HttpProcessor实例的assign方法，并将客户端socket对象作为参数传入：

```
processor.assign(socket); 
```

这时，HttpProcessor实例开始读取socket的输入流，解析http请求。这里有一个重点，assign方法必须立刻返回，不能等待HttpProcessor实例完成解析再返回，这样才能处理后续的http请求。由于每个HttpProcessor都可以使用它自己的线程进行处理，所以这并不难实现。

## HttpProcessor类

HttpProcessor类与第三章中的实现相类似。本章讨论下它的assign方法是如何实现异步功能的（即可同时处理多个http请求）。

在第三章中，HttpConnector类运行在其自己的线程中。在处理下一个请求之前，它必须等待当前请求的处理完成。

### 第三章中的HttpConnector类的run方法

```java
public void run() {   
      ...   
     while (!stopped) {   
       Socket socket = null;   
       try {   
         socket = serversocket.accept();   
       }        catch (Exception e) {   
         continue;   
       }   
       // Hand this socket off to an Httpprocessor   
       HttpProcessor processor = new Httpprocessor(this);   
       processor.process(socket);   //同步的！！！
     }   
   } 
```

process方法是同步的。

### 默认的HttpConnector类的run方法

```java
public void run() {   
      ...   
     while (!stopped) {   
       Socket socket = null;   
       try {   
         socket = serversocket.accept();   
       }        catch (Exception e) {   
         continue;   
       }   
       // Hand this socket off to an Httpprocessor   
       HttpProcessor processor = new Httpprocessor(this);   
       processor.start();
       processor.assign(socket);   
     }   
   } 
```

但是在tomcat的默认连接器中，HttpProcessor实现了java.lang.Runnable接口，每个HttpProcessor的实例都可以在其自己的线程中运行，成为“处理器线程”（“processor thread”）。HttpConnector创建每个HttpProcessor实例时，都会调用其start方法，启动其处理器线程。

### 默认的HttpProcessor实例的run方法

```java
public void run() {   
  // Process requests until we receive a shutdown signal   
  while (!stopped) {   
    // Wait for the next socket to be assigned   
    Socket socket = await();     //阻塞了！！！
    if (socket == null)   
      continue;   
    // Process the request from this socket   
    try {   
      process(socket);   
    } catch (Throwable t) {   
      log("process.invoke", t);   
    }   
    // Finish up this request   
    connector.recycle(this);   
  }   
  // Tell threadStop() we have shut ourselves down successfully   
  synchronized (threadSync) {   
    threadSync.notifyAll();   
  }   
}  
```

这个循环体做的事是：[获取socket]()，[进行处理]()，[调用connector的recycle方法将当前的HttpProcessor入栈]()。

> 注意，循环体在执行到await方法时会暂停当前处理器线程的控制流，直到获取到一个新的socket。换句话说，在HttpConnector调用HttpProcessor实例的assign方法前，HttpProcessor在await()会一直等下去。但是，assign方法并不是在当前线程中执行的，而是在HttpConnector的run方法中被调用的。这里称HttpConnector实例所在的线程为连接器线程（connector thread）。

### Connector和Processor线程互相通知对方

那么，assign方法是如何通知await方法它已经被调用了呢？方法是使用一个成为available的boolean变量和java.lang.Object的wait和notifyAll方法。

> 注意，wait方法会暂停本对象所在的当前线程，使其处于等待状态，直到另一线程调用了该对象的notify或notifyAll方法。

#### Processor的assign()方法

> 在Connector的run()中被调用,processor.assign(socket)

```java
synchronized void assign(Socket socket) { 
   // Wait for the processor to get the previous socket 
   while (available) { 
     try { 
       wait(); 
     } 
     catch (InterruptedException e) { 
     } 
   } 
   // Store the newly available Socket and notify our thread 
   this.socket = socket; 
   available = true; 
   notifyAll(); 
   ... 
}
```

#### Processor的await()方法

> 在Connector的run()中被调用,processor.start()

```java
private boolean available = false;

private synchronized Socket await() { 
   // Wait for the Connector to provide a new Socket 
    while (!available) { 
     try { 
       wait(); 
     } 
     catch (InterruptedException e) { 
     } 
   } 
 
   // Notify the Connector that we have received this Socket 
   Socket socket = this.socket; 
   available = false; 
   notifyAll(); 
   return (socket); 
}
```

当处理器线程刚刚启动时，available值为false，线程在循环体内wait，直到任意一个线程调用了notify或notifyAll方法。也就是说，调用wait方法会使线程暂定，直到连接器线程调用HttpProcessor实例的notify或notifyAll方法。

当一个新socket被设置后，连接器线程调用HttpProcessor的assign方法。此时available变量的值为false，会跳过循环体，该socket对象被设置到HttpProcessor实例的socket变量中。然后连接器变量设置了available为true，调用notifyAll方法，唤醒处理器线程。此时available的值为true，跳出循环体，将socket对象赋值给局部变量，将available设置为false，调用notifyAll方法，并将给socket返回。

#### 问题

- 为什么await方法要使用一个局部变量保存socket对象的引用，而不返回实例的socket变量呢？是因为在当前socket被处理完之前，可能会有新的http请求过来，产生新的socket对象将其覆盖。 <!--线程安全：使用栈内的局部变量，保证线程独有。否则，对象的局部变量socket（句柄），可能会被重新赋值，指向堆中的新的socket对象-->

- 为什么await方法要调用notifyAll方法？考虑这种情况，当available变量的值还是true时，有一个新的socket达到。在这种情况下，连接器线程会在assign方法的循环体中暂停，直到处理器线程调用notifyAll方法。