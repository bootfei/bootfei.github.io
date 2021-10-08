---
title: chapter05-servlet容器
date: 2021-10-07 20:14:36
tags:
---

https://blog.csdn.net/lovejavaydj/category_9267782.html



servelet容器是用来处理请求servlet资源，为web client端填充response对象的模块。servlet容器是org.apache.catalina.Container接口的实现。

<!--org.apache.catalina包是接口, -org.apache.catalina.core包是具体实现-->



## 5.1 Container接口

一个容器必须实现 org.apache.catalina.Container 接口。如第4章中看到，传递一个 Container 实例给 Connector 对象的 setContainer()方法，然后Connector 就可以调用 container 的 invoke() 方法，重新看第4章中Bootstrap 类的代码如下：

```java
HttpConnector connector = new HttpConnector();
SimpleContainer container = new SimpleContainer();
connector.setContainer(container);
```

首先需要注意的是，对于 Catalina 容器，在不同的概念上，它一共有4种不同类型的容器：

1》Engine：表示整个 Catalina 的 servlet 引擎
2》Host：表示一拥有数个上下文(context)的虚拟主机
3》Context：表示一 Web 应用，一个 context 包含一个或多个wrapper
4》Wrapper：表示一个独立的 servlet

上面的每个概念级别都由org.apache.catalin包中的接口表示。Engine、Host、Context和 Wrapper 接口都实现了 Container 接口。它们的标准实现是 StandardEngine,StandardHost, StandardContext, StandardWrapper，它们都是org.apache.catalina.core 包的一部分。

图 5.1 表示了 Container 接口和它的子接口的结构图。注意接口都是org.apache.catalina 包的，而所有的类都是 org.apache.catalina.core 包的。
![这里写图片描述](https://img-blog.csdn.net/20170115205718774?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG92ZUphdmFZREo=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图 5.1: Container相关类图

> 注意:
>
> 所有的类都扩展自抽象类 ContainerBase。
>
> Catalina 功能部署不一定需要所有的四种类型容器。例如本章第一个应用程序就仅包括一个 wrapper，而第二个应用程序包含 Context 和wrapper 容器模块。在本章附带的应用程序中不需要host和engine。

一个容器可以有一个或多个低层次上的子容器。例如，一个 Context 有一个或多个 wrapper；一个host有零个或多个context。 然而 wrapper 作为最底层容器，则不能包含子容器。把个容器添加到另一容器中可以使用 Container 接口中定义的 addChild()方法

```java
public interface Container {

    // 添加子容器；Host容器下只能添加Context容器。。。
    void addChild(Container child);

    // 移除子容器
    void removeChild(Container child);

    // 根据名称查找子容器
    public Container findChild(String name);

    // 查找子容器集合
    public Container[] findChildren();

    // 其它组件的get/set方法，包括：载入器（Loader）、记录器（Logger）、Session管理器（Manager）、领域（Realm）、资源（Resource）

    容器是Tomcat的核心，所以才将所有组件都与容器连接起来，而且通过Lifecyle接口，使我们可以只启动容器组件就可以了（他帮我们启动其它组件）
}
```

> 注：这是组合模式的一种使用

一个容器还包含一系列的部分如 Lodder、Loggee、Manager、Realm 和Resources。我们将会在后边章节中讨论这些组成部分。

更有意思的是Container接口被设计成Tomcat管理员可以通过server.xml文件配置来决定其工作方式的模式。它通过一个 pipeline和容器中一系列的valves来实现，这些内容将会在下一节 “管道流水线任务”中讨论。

## 5.2 管道任务

本章节介绍当connector 调用容器(container)的 invoke() 方法会发生什么。后续子章节中讨论org.apache.catalina 中4个相关接口：Pipeline, Valve, ValveContext和Contained。

一个管道(pipeline)中包含了该容器要调用的所有任务。每一个阀门(valve)表示着一特定任务。一个容器的管道中有一个<font color="red">基本的阀门</font>，但是我们可以添加任意想要添加的阀门。阀门的数目定义为添加的阀门的个数（不包括基本阀门）。有趣的是，阀门可以通过编辑 Tomcat 的配置文件 server.xml 来动态地添加。

一个管道线就像一个过滤链，每一个阀门像一个过滤器。跟过滤器一样，一个阀门可以操作处理传递给它的 request 和 response 对象。一个阀门完成处理后，它则进一步调用管道中的下一个阀门，基本阀门总是在最后才被调用。

一个容器可以有一个管道。当容器的 invoke() 方法被调用时，容器将通过管道处理，且管理调用在其中的第一个阀门，一个接一个阀门的调用处理，直到所有阀门都被处理完毕。可以想象管道的 invoke() 方法的伪代码如下所示：

```java
// invoke each valve added to the pipeline
for (int n=0; n<valves.length; n++) {
    valve[n].invoke( ... );
}

// then, invoke the basic valve
basicValve.invoke( ... );
```

但是，Tomcat 设计者通过引入org.apache.catalina.ValveContext接口选择了一种不同处理方式。这里将介绍它是如何工作的。

[容器不会硬编码它的invoke()方法被调用时应该做什么。反而，容器调用的是管道的 invoke()方法]()。管道接口的 invoke() 方法跟容器接口的invoke() 方法签名相同，方法签名如下：

```
public void invoke(Request request, Response response) throws IOException, ServletException;
```

这里是 Container 接口中 invoke() 方法在org.apache.catalina.core.ContainerBase 的实现：

```java
public void invoke(Request request, Response response) throws IOException, ServletException {
    pipeline.invoke(request, response); //这里pipeline是容器中 Pipeline 接口的一个实例。
}
```



[现在，管道必须保证添加给它的阀门必须如基本阀门一样被调用一次。]()**管道通过创建一**
**个 ValveContext 接口的实例来实现**。ValveContext 是管道的内部类，这样 ValveContext 就可以访问管道中所有成员。ValveContext 中最重要的方法是 invokeNext() 方法：

在创建一个 ValveContext 实例之后，管道调用 ValveContext 的 invokeNext()方法。ValveContext 会先唤起管道中的第一个阀门，然后第一个阀门会在完成它的任务之前继续唤起下一个阀门。ValveContext 将它自己传递给每一个阀门，那么该阀门就可以调用 ValveContext 的 invokeNext() 方法。Valve 接口的 invoke()签名如下：

```java
public void  invoke(Request request, Response response,
    ValveContext ValveContext) throws IOException, ServletException
```

一个Valve的 invoke() 方法可以如下实现：

```java
public void invoke(Request request, Response response,
    ValveContext valveContext) throws IOException, ServletException {
    // Pass the request and response on to the next valve in our pipeline
    valveContext.invokeNext(request, response);
    // now perform what this valve is supposed to do
    ...
}
```

org.apache.catalina.core.StandardPipeline 类是所有容器中Pipeline的实现。在Tomcat4 中，这个类中有一个[内部类 StandardPipelineValveContext]() 实现了ValveContext 接口，

Listing 5.1: Tomcat 4中的StandardPipelineValveContext 类

```java
//pipeline的内部类
protected class StandardPipelineValveContext implements ValveContext {
    protected int stage = 0;
    public String getInfo() {
        return info;
    }
    public void invokeNext(Request request, Response response)
        throws IOException, ServletException {
        int subscript = stage;
        stage = stage + 1;
        // Invoke the requested Valve for the current request thread
        if (subscript < valves.length) {
            valves[subscript].invoke(request, response, this);
        }else if ((subscript == valves.length) && (basic != null)) {
            basic.invoke(request, response, this);
        }else {
            throw new ServletException (sm.getString("standardPipeline.noValve"));
        }
    }
}

```

invokeNext()方法中使用subscript和stage记住哪个阀门被唤醒。当第一次唤醒时，subscript的值是 0，stage的值是 1。所以，第一个阀门(数组下标是0)被唤醒，管道的阀门获得 ValveContext 实例接收到ValveContext实例并调用它的 invokeNext() 方法。这时subscript的值是 1， 所以第二个阀门被唤醒，然后一步步地这样继续进行。<!--疑惑，stage作为类变量，是线程安全的吗-->

当invokeNext()在最后一个阀门中调用时，subscript值等于总阀门的个数。因此这时，基本阀门被唤醒调用。

### **5.2.1 Pipeline接口**

我们提到的Pipeline接口的第一种方法是invoke()方法，容器调用它来开始调用管道中的阀门和基本阀门。Pipeline接口允许我们通过addValve()方法添加一个新的阀门或者通过removeValve()方法删除一个阀门。最后，可以使用 setBasic()方法来分配一个基本阀门给管道，使用getBasic()方法会得到基本阀门。最后调用的基本阀门负责处理请求和相应的响应。

Listing 5.3: Pipeline接口

```
package org.apache.catalina;
import java.io.IOException;
import javax.servlet.ServletException;

public interface Pipeline {
    public Valve getBasic();
    public void setBasic(Valve valve);
    public void addValve(Valve valve);
    public Valve[] getValves();
    public void invoke(Request request, Response response)
        throws IOException, ServletException;
    public void removeValve(Valve valve);
}
```

### **5.2.2 Valve接口**

Value接口表示一个阀门，该组件负责处理请求。该接口有两个方法，invoke() 和getInfo()方法。invoke()方法如上面已讨论过，getInfo()方法返回阀门的信息。

```
package org.apache.catalina;
import java.io.IOException;
import javax.servlet.ServletException;

public interface Valve {
    public String getInfo();
    public void invoke(Request request, Response response,
        ValveContext context) throws IOException, ServletException;
}123456789
```

### **5.2.3 ValveContext接口**

ValveContext接口有两个方法，invokeNext()方法如上已讨论，getInfo()方法会返回valveContext的实现信息。

```
package org.apache.catalina;
import java.io.IOException;
import javax.servlet.ServletException;

public interface ValveContext {
    public String getInfo();
    public void invokeNext(Request request, Response response)
        throws IOException, ServletException;
}123456789
```

### **5.2.4 Contained接口**

阀门类可以选择性实现org.apache.catalina.Contained接口。此接口指定实现类最多与一个相关联容器实例。

```
package org.apache.catalina;

public interface Contained {
    public Container getContainer();
    public void setContainer(Container container);
}123456
```



## 5.3 Wrapper接口

org.apache.catalina.Wrapper 接口表示一个包装器。包装器是表示单个servlet定义的容器。包装器继承了Container接口，并且添加了几个方法。包装器的实现类负责管理其servlet 的生命中期，包括 servlet 的init()、service()、和 destroy()方法。由于包装器是最底层的容器，所以不可以将子容器添加给它。[如果 addChild()方法被调用，则会产生IllegalArgumantException 异常。]()

包装器接口中重要方法有 allocate() 和 load() 方法。allocate() 方法负责定位该包装器表示的 servlet 的实例。allocate()方法必须考虑一个 servlet 是否实现了javax.servlet.SingleThreadModel 接口，该部分内容将会在 11 章中进行讨论。load() 方法负责加载和初始化 servlet 实例。

```java
// 载入并加载servlet，在它的子类StandardWrapper中直接调用的loadServlet方法
void load() throws ServletException;

// 该方法会返回已加载的servlet类，还要考虑它是否实现了SingleThreadModel
Servlet allocate() throws ServletException;
```

## **5.4 Context接口**

一个 context 容器表示一个 web 应用。一个 context 通常含有一个或多个包装器作为其子容器。重要的方法包括 addWrapper(), createWrapper() 等方法。该接口将会在第 12 章中详细介绍。

## **5.5 Wrapper应用Demo**

这个应用Demo展示了如何写一个简单的容器模型。该应用程序的核心类是ex05.pyrmont.core.SimpleWrapper，它实现了 Wrapper 接口。SimpleWrapper类包括一个 Pipeline（由 ex05.pyrmont.core.SimplePipeline 实现）和一个Loader 类（ex05.pyrmont.core.SimpeLoader）来加载一个 servlet。管道包括一个基本阀门（ex05.pyrmont.core.SimpleWrapperValve）和两个另外的阀门
(ex05.pyrmont.core.ClientIPLoggerValve 和ex05.pyrmont.core.HeaderLoggerValve)。该应用的类结构图如图 5.3 所示：

![这里写图片描述](https://img-blog.csdn.net/20170115210024548?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG92ZUphdmFZREo=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


注意：该容器使用的是Tomcat 4的默认连接器

包装器包装的是前面章节已经使用过的 ModernServlet类。这个应用程序表示一个 servlet 容器可以只有一单一的包装器构成。这些类都没有完整的实现，只是实现了必要方法。接下来看程序的具体实现。

#### **5.5.1 core.SimpleLoader**

容器中加载 servlet 的任务分配给了 Loader 实现。在该程序中 SimpleLoader就是一个 Loader 实现。它知道如何定位一个 servlet，并且通过 getClassLoader()获得一个 java.lang.ClassLoader 实例用来查找 servlet 类位置。

- SimpleLoader定义了 3 个变量

```java
//第一个是 WEB_ROOT 用来指明在哪里查找 servlet 类。
public static final String WEB_ROOT =
        System.getProperty("user.dir") + File.separator + "webroot";
//另外两个变量是 ClassLoader 和 Container：
ClassLoader classLoader = null;
Container container = null;
```

- SimpleLoader 类的构造器初始化类加载器，以便于准备返回一个 SimpleWrapper实例。

```java
public SimpleLoader() {
    try {
        URL[] urls = new URL[l];
        URLStreamHandler streamHandler = null;
        File classPath = new File(WEB_ROOT);
        String repository = (new URL("file", null,
            classPath.getCanonicalPath() + File.separator)).toString() ;
        urls[0] = new URL(null, repository, streamHandler);
        classLoader = new URLClassLoader(urls);
    }catch (IOException e) {
        System.out.println(e.toString() );
    }
}
```

该程序的构造器用于初始化一个类加载器如前面章节所用的一样。container变量表示容器跟该加载器是相关联。

注意：加载器将在第8章详细讨论。

#### **5.5.2 core.SimplePipeline**

SimplePipeline 实现了 org.apache.catalina.Pipeline 接口。该类中最重要的方法是 invoke() 方法，其中包括了一个内部类 SimplePipelineValveContext。SimplePipelineValveContext 实现了 org.apache.catalina.ValveContext 接口如上面章节所介绍。

#### **5.5.3 core. SimpleWrapper**

该类实现了 org.apache.catalina.Wrapper 接口并且实现了 allocate() 方法和load() 方法，并声明了如下变量：

```
private Loader loader;
protected Container parent = null;12
```

loader 变量用于加载一个 servlet 类。parent 变量表示该包装器的父容器。这意味着，该容器可以是其它容器的子容器，例如 Context。

需要特别注意 getLoader()方法，

```java
public Loader getLoader() {
    if (loader != null)
        return (loader);
    if (parent != null)
        return (parent.getLoader());
    return (null);
}1234567
```

getLoader()方法用于返回一个 Loader 对象用于加载一个 servlet 类。如果一个包装器跟一个加载器相关联，会返回该加载器。否则返回其父容器的加载器，如果没有父容器，则返回 null。

SimpleWrapper 类有一个管道和该管道的基本阀门。这些工作在SimpleWrapper 的构造函数中完成。

Listing 5.8:

```
public SimpleWrapper() {
    pipeline.setBasic(new SimpleWrapperValve());
}
```

其中，pipeline是 SimplePipeline 类的一个实例：

```
private SimplePipeline pipeline = new SimplePipeline(this);
```

#### **5.5.4 core. SimpleWrapperValve**

SimpleWrapperValve 类是一个给 SimpleWrapper 类专门处理请求的基本阀门。它实现了 org.apache.catalina.Valve 接口和org.apache.catalina.Contained接口。最重要的方法是 invoke() 方法，如 Listing5.9 所示：

Listing 5.9: SimpleWrapperValve类的invoke()方法：

```
public void invoke(Request request, Response response, ValveContext valveContext)
    throws IOException, ServletException {

    SimpleWrapper wrapper = (SimpleWrapper) getContainer();
    ServletRequest sreq = request.getRequest();
    ServletResponse sres = response.getResponse();
    Servlet servlet = null;
    HttpServletRequest hreq = null;
    if (sreq instanceof HttpServletRequest)
      hreq = (HttpServletRequest) sreq;
    HttpServletResponse hres = null;
    if (sres instanceof HttpServletResponse)
      hres = (HttpServletResponse) sres;

    // Allocate a servlet instance to process this request
    try {
      servlet = wrapper.allocate();
      if (hres!=null && hreq!=null) {
        servlet.service(hreq, hres);
      }
      else {
        servlet.service(sreq, sres);
      }
    }
    catch (ServletException e) {
    }
  }123456789101112131415161718192021222324252627
```

由于 SimpleWrapperValve 被当做一基本阀门来使用，所以它的 invoke() 方法不需要invokeNext()方法。invoke()方法调用SimpleWrapper的allocate()方法获得servlet的实例。然后调用 servlet 的 service() 方法。注意包装器管道的基本阀门唤醒的是 servlet 的 service ()方法，而不是 wrapper自己。

#### **5.5.5 ex05.pyrmont. valves. ClientIPLoggerValve**

ClientIPLoggerValve 是一个阀门，它打印客户端的 IP 地址到控制台。该类如 Listing5.10:

Listing 5.10: ClientIPLoggerValve类

```
package ex05.pyrmont.valves;

import java.io.IOException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletException;
import org.apache.catalina.Request;
import org.apache.catalina.Response;
import org.apache.catalina.Valve;
import org.apache.catalina.ValveContext;
import org.apache.catalina.Contained;
import org.apache.catalina.Container;


public class ClientIPLoggerValve implements Valve, Contained {

  protected Container container;

  public void invoke(Request request, Response response, ValveContext valveContext)
    throws IOException, ServletException {

    // Pass this request on to the next valve in our pipeline
    valveContext.invokeNext(request, response);
    System.out.println("Client IP Logger Valve");
    ServletRequest sreq = request.getRequest();
    System.out.println(sreq.getRemoteAddr());
    System.out.println("------------------------------------");
  }

  public String getInfo() {
    return null;
  }

  public Container getContainer() {
    return container;
  }

  public void setContainer(Container container) {
    this.container = container;
  }
}12345678910111213141516171819202122232425262728293031323334353637383940
```

注意 invoke() 方法，它的第一件事情是调用阀门上下文 invokeNext ()方法来唤醒下
一个阀门，然后它会打印出请求对象的 getRemoteAddr() 方法的输出。

#### **5.5.6 ex05.pyrmont. valves. HeaderLoggerValve**

该类跟 ClientIPLoggerValve 类非常相似。HeaderLoggerValve 是一个阀门打印请求头部信息到控制台上。该类如 Listing5.11：

Listing 5.11: HeaderLoggerValve 类

```
package ex05.pyrmont.valves;

import java.io.IOException;
import java.util.Enumeration;
import javax.servlet.ServletRequest;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import org.apache.catalina.Request;
import org.apache.catalina.Response;
import org.apache.catalina.Valve;
import org.apache.catalina.ValveContext;
import org.apache.catalina.Contained;
import org.apache.catalina.Container;


public class HeaderLoggerValve implements Valve, Contained {

  protected Container container;

  public void invoke(Request request, Response response, ValveContext valveContext)
    throws IOException, ServletException {

    // Pass this request on to the next valve in our pipeline
    valveContext.invokeNext(request, response);

    System.out.println("Header Logger Valve");
    ServletRequest sreq = request.getRequest();
    if (sreq instanceof HttpServletRequest) {
      HttpServletRequest hreq = (HttpServletRequest) sreq;
      Enumeration headerNames = hreq.getHeaderNames();
      while (headerNames.hasMoreElements()) {
        String headerName = headerNames.nextElement().toString();
        String headerValue = hreq.getHeader(headerName);
        System.out.println(headerName + ":" + headerValue);
      }

    }
    else
      System.out.println("Not an HTTP Request");

    System.out.println("------------------------------------");
  }

  public String getInfo() {
    return null;
  }

  public Container getContainer() {
    return container;
  }

  public void setContainer(Container container) {
    this.container = container;
  }
}12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455
```

注意其 invoke() 方法，该方法首先调用阀门的 invokeNext()方法唤醒下一个阀门。然后打印出头部的值。

#### **5.5.7 pyrmont.startup.Bootstrap1**

Bootstrap1 用于启动这个应用程序。

Listing 5.12: Bootstrap1 类

```java
package ex05.pyrmont.startup;

import ex05.pyrmont.core.SimpleLoader;
import ex05.pyrmont.core.SimpleWrapper;
import ex05.pyrmont.valves.ClientIPLoggerValve;
import ex05.pyrmont.valves.HeaderLoggerValve;
import org.apache.catalina.Loader;
import org.apache.catalina.Pipeline;
import org.apache.catalina.Valve;
import org.apache.catalina.Wrapper;
import org.apache.catalina.connector.http.HttpConnector;

public final class Bootstrap1 {
  public static void main(String[] args) {

/* call by using http://localhost:8080/ModernServlet,
   but could be invoked by any name */

    HttpConnector connector = new HttpConnector();
    Wrapper wrapper = new SimpleWrapper();
    wrapper.setServletClass("ModernServlet");
    Loader loader = new SimpleLoader();
    Valve valve1 = new HeaderLoggerValve();
    Valve valve2 = new ClientIPLoggerValve();

    wrapper.setLoader(loader);
    ((Pipeline) wrapper).addValve(valve1);
    ((Pipeline) wrapper).addValve(valve2);

    connector.setContainer(wrapper);

    try {
      connector.initialize();
      connector.start();

      // make the application wait until we press a key.
      System.in.read();
    }
    catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

创建 HttpConnector 和 SimpleWrapper 类的实例后，分配ModernServlet 给 SimpleWrapper 的 setServletClass() 方法，告诉包装器要加载的类的名字以便于加载。

```
wrapper.setServletClass("ModernServlet");1
```

然后创建了加载器和两个阀门，然后将其加载器赋给包装器：

```
Loader loader = new SimpleLoader();
Valve valve1 = new HeaderLoggerValve();
Valve valve2 = new ClientIPLoggerValve();
wrapper.setLoader(loader);1234
```

然后把两个阀门添加到包装器管道中：

```
((Pipeline) wrapper).addValve(valve1);
((Pipeline) wrapper).addValve(valve2);12
```

最后，把包装器当做容器添加到连接器中，然后初始化并启动连接器：

```
connector.setContainer(wrapper);
try {
    connector.initialize();
    connector.start();1234
```

下一行允许用户在控制台键入回车键以停止程序。

```
// make the application wait until we press Enter.
System.in.read();12
```

#### **5.5.8 运行Demo**

在 windows 下，可以在工作目录下面如下运行该程序：

```
java -classpath ./lib/servlet.jar;./ ex05.pyrmont.startup.Bootstrap11
```

在 Linux 下，使用冒号分开两个库：

```
java -classpath ./lib/servlet.jar:./ ex05.pyrmont.startup.Bootstrap11
```

可以使用下面的 URL 来请求servlet：

```
http://localhost:80801
```

浏览器将会显示从 ModernServlet 得到的响应回复。跟下面相似的内容会显示在控制台上：

```
ModernServlet -- init
Client IP Logger Valve
127.0.0.1
------------------------------------
Header Logger Valve
host:localhost:8080
user-agent:Mozilla/5.0 (Windows NT 10.0; WOW64; rv:50.0) Gecko/20100101 Firefox/50.0
accept:text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
accept-language:zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
accept-encoding:gzip, deflate
connection:keep-alive
upgrade-insecure-requests:1
------------------------------------
```

## **5.6 Context应用Demo**

在本章第一个Demo中，介绍了如何部署一个仅仅包括一个包装器(Wrapper)的简单web应用。该程序仅包括一个 servlet。也许会有一些应用仅仅需要一个 servlet，可是大多数的网络应用需要多个 servlet。在这些应用中，我们需要一个跟包装器（wrapper）不同的容器：上下文（context）。

第二个Demo将会示范如何使用一个包含两个包装器的上下文来包装两个servlet 类。当有多于一个包装器时，需要一个 map 来处理这些子容器——本Demo中使用Context容器，对于特殊的请求可以使用特殊的子容器来处理。

> 注意 使用 map 方法是在 Tomcat4 中，Tomcat 5 使用了另一种机制来查找子容器。

在这个程序中，mapper 是 ex05.pyrmont.core.SimpleContextMapper 类的一个实例，它继承Tomcat 4 中org.apache.catalina.Mapper 接口。一个容器也可以有多个 mapper 来支持多协议。例如容器可以用一个 mapper 来支持 HTTP 协议，而使用另一个 mapper 来支持 HTTPS 协议。Listing5.13 提供了 Tomcat4 中的 Mapper 接口。

```
package org.apache.catalina;

public interface Mapper {
    public Container getContainer();
    public void setContainer(Container container);
    public String getProtocol();
    ublic void setProtocol(String protocol);
    ublic Container map(Request request, boolean update);
}
```

getContainer()获取该容器的 mapper，setContainer() 方法用于关联一个容器到mapper。getProtocol() 返回该 mapper 负责处理的协议，setProtocol()用于分配该容器要处理的协议。map()方法返回处理一个特殊请求的子容器。
![这里写图片描述](https://img-blog.csdn.net/20170115210322558?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG92ZUphdmFZREo=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图 5.4 是该Demo结构图。

SimpleContext表示一个上下文，它使用SimpleContextMapper作为它的mapper，SimpleContextValve 作为它的基本阀门。该上下文包括两个阀门ClientIPLoggerValve 和 HeaderLoggerValve。用 SimpleWrapper 表示的两个包装器作为该上下文的子容器被添加到其中。包装器 SimpleWrapperValve 作为它的基阀门，但是没有其它的阀门了。

该Context应用程序使用同一个加载器、两个阀门。[但是加载器和阀门是跟该Context关联的，而不是跟包装器关联]()。这样，两个包装器就可以都使用该加载器。该Conetext被当做连接器的容器。因此，连接器每次收到一个 HTTP 请求可以使用上下文的 invoke() 方法。根据前面介绍的内容，其余的不难理解。

1》一个容器有一个管道，容器的 invoke() 方法会调用管道的invoke()方法
2》管道的 invoke()方法会调用添加到容器中的阀门的 invoke()方法，然后调用基本阀门的 invoke()方法
3》在一包装器中，基本阀门负责加载相关的 servlet 类并对请求作出相应
4》在一个有子容器的上下文中，基阀门使用 mapper 来查找负责处理请求的子容器。如果一个子容器被找到，子容器的 invoke() 方法会被调用，然后返回步骤 1

现在让我们看看实现中的处理流程。

SimpleContext 的 invoke() 方法调用管道的 invoke() 方法：

```
public void invoke(Request request, Response response)
    throws IOException, ServletException {
        pipeline.invoke(request, response);
}1234
```

pipeline是SimplePipeline 类实例，用来表示管道，它的 invoke()方法如下所示：

```
public void invoke(Request request, Response response)
    throws IOException, ServletException {
    // Invoke the first Valve in this pipeline for this request
    (new SimplePipelineValveContext()).invokeNext(request, response);
}12345
```

如“管道任务”一节中介绍的，该段代码唤醒所有阀门，然后调用基阀门的invoke()方法。在SimpleContext中SimpleContextValve代表着基阀门。在它的invoke()方法中SimpleContextValve使用上下文的mapper是查找一个包装器：

```
// Select the Wrapper to be used for this Request
Wrapper wrapper = null;
try {
    wrapper = (Wrapper) context.map(request, true);
}12345
```

如果一个包装器被找到，它的invoke()方法会被调用。

```
wrapper.invoke(request, response);1
```

本Demo着通过SimpleWrapper表示一个包装器。如下是其invoke()方法，和SimpleContext类的invoke()方法完全一样。

```
public void invoke(Request request, Response response)
    throws IOException, ServletException {
    pipeline.invoke(request, response);
}1234
```

管道是SimplePipeline的一个实例，其调用方法已在上面列出。本Demo中包装器是SimpleWrapperValve的实例，它除了有基阀门外没其它阀门。包装器的管道调用SimpleWrapperValve类的invoke()方法，它分配一个servlet并调用其service()方法，如上文“Wrapper应用Demo”一节中所述。

注意，包装器不与加载器相关联，而是上下文与其关联。 因此，SimpleWrapper类的getLoader()方法返回父级（Context）的加载器。

有4个类：SimpleContext, SimpleContextValve, SimpleContextMapper和Bootstrap2在前面小节没被提到，在下面将讨论。

#### **5.6.1 core.SimpleContextValve**

此类作为SimpleContext的基阀门。它的最重要的方法invoke()代码如Listing 5.14：

Listing 5.14: SimpleContextValve类的invoke()方法：

```java
public void invoke(Request request, Response response, ValveContext valveContext)
    throws IOException, ServletException {
    // Validate the request and response object types
    if (!(request.getRequest() instanceof HttpServletRequest) ||
      !(response.getResponse() instanceof HttpServletResponse)) {
      return;     // NOTE - Not much else we can do generically
    }

    // Disallow any direct access to resources under WEB-INF or META-INF
    HttpServletRequest hreq = (HttpServletRequest) request.getRequest();
    String contextPath = hreq.getContextPath();
    String requestURI = ((HttpRequest) request).getDecodedRequestURI();
    String relativeURI =
      requestURI.substring(contextPath.length()).toUpperCase();

    Context context = (Context) getContainer();
    // Select the Wrapper to be used for this Request
    Wrapper wrapper = null;
    try {
      wrapper = (Wrapper) context.map(request, true);
    }
    catch (IllegalArgumentException e) {
      badRequest(requestURI, (HttpServletResponse) response.getResponse());
      return;
    }
    if (wrapper == null) {
      notFound(requestURI, (HttpServletResponse) response.getResponse());
      return;
    }
    // Ask this Wrapper to process this Request
    response.setContext(context);
    wrapper.invoke(request, response);
  }
```

#### **5.6.2 core.SimpleContextMapper**

SimpleContextMapper类实现Tomcat4中的org.apache.catalina.Mapper接口，并被设计成和SimpleContext实例相关联。

Listing 5.15: The SimpleContext 类

```java
public class SimpleContextMapper implements Mapper {
  private SimpleContext context = null;

  public Container getContainer() {
    return (context);
  }

  public void setContainer(Container container) {
    if (!(container instanceof SimpleContext))
      throw new IllegalArgumentException
        ("Illegal type of container");
    context = (SimpleContext) container;
  }

  public String getProtocol() {
    return null;
  }

  public void setProtocol(String protocol) {
  }


  /**
   * Return the child Container that should be used to process this Request,
   * based upon its characteristics.  If no such child Container can be
   * identified, return <code>null</code> instead.
   */
  public Container map(Request request, boolean update) {
    // Identify the context-relative URI to be mapped
    String contextPath =
      ((HttpServletRequest) request.getRequest()).getContextPath();
    String requestURI = ((HttpRequest) request).getDecodedRequestURI();
    String relativeURI = requestURI.substring(contextPath.length());
    // Apply the standard request URI mapping rules from the specification
    Wrapper wrapper = null;
    String servletPath = relativeURI;
    String pathInfo = null;
    String name = context.findServletMapping(relativeURI);
    if (name != null)
      wrapper = (Wrapper) context.findChild(name);
    return (wrapper);
  }
}
```

如果我们传递一个不是SimpleContext实例的容器给setContainer()方法时，它将抛出IllegalArgumentException异常。map()方法返回一个子容器（wrapper），它负责处理请求。map()方法接收二个参数，一个是请求对象，一个是布尔值。这里的方法实现忽略了第二个参数。该方法执行的操作是检索从请求对象中得到上下文路径，并使用context的findServletMapping()方法获取与其路径关联的名称。 如果找到名称，它使用context的findChild()方法来获取Wrapper的实例。

**5.6.3 ex05.pyrmont.core.SimpleContext**

SimpleContext类是本Demo中Context一个实现。它是分配给连接器的主容器。然而，每个单独的servlet的处理由包装器执行。本应用Demo有2个servlet：PrimitiveServlet和ModernServlet，如此一来就有2个包装器。每个包装器都有名字。名称为Primitive是PrimitiveServlet的包装器，Modern是ModernServlet的包装器。对于每个请求，SimpleContext要决定调用哪个包装器，这个必须使用包装器的名称映射匹配请求URL。在此应用程序中，我们有两个可用于调用2个包装器的URL模式。第一个模式是/ Primitive，它映射到Primitive包装器。 第二个模式是/ Modern，它被映射到Modern包装器。 当然，对于给定的servlet，我们可以使用多个模式。 我们只需要添加这些模式即可。

从Container和Context接口有很多方法，SimpleContext必须实现。 大多数方法是留白，但是这些与映射相关的方法给予了实现代码。 这些方法如下：

1》addServletMapping()方法——添加URL/包装器名称映射对。 添加的每一个，可用于调用具有给定名称的包装器
2》findServletMapping()方法——获取和URL对应的包装器名称。这个方法用于为一个指定的URL查找哪个包装器应该被调用。如果给定的模式之前未通过addServletMapping()方法添加过，那么此方法将返回null
3》addMapper()方法——添加一个mapper给context。SimpleContext申明mapper和mappers变量。mapper为默认的mapper，mappers包含SimpleContext实例中所有mapper。第一个添加到SimpleContext中的mapper将作为默认mapper
4》findMapper()方法——查找当前mapper。在SimpleContext中，返回默认mapper。
5》map()方法——返回负责处理此请求的包装器

此外SimpleContext也提供了addChild(),findChild(), 和findChildren()方法实现。addChild()用于添加一个wrapper给context；findChild()用于获取指定名称的wrapper；findChildren()返回SimpleContext实例当中的所有包装器。

**5.6.4 ex05.pyrmont.startup.Bootstrap2**

Listing 5.16: The Bootstrap2 类

```
package ex05.pyrmont.startup;

import ex05.pyrmont.core.SimpleContext;
import ex05.pyrmont.core.SimpleContextMapper;
import ex05.pyrmont.core.SimpleLoader;
import ex05.pyrmont.core.SimpleWrapper;
import ex05.pyrmont.valves.ClientIPLoggerValve;
import ex05.pyrmont.valves.HeaderLoggerValve;
import org.apache.catalina.Context;
import org.apache.catalina.Loader;
import org.apache.catalina.Mapper;
import org.apache.catalina.Pipeline;
import org.apache.catalina.Valve;
import org.apache.catalina.Wrapper;
import org.apache.catalina.connector.http.HttpConnector;

public final class Bootstrap2 {
  public static void main(String[] args) {
    HttpConnector connector = new HttpConnector();
    Wrapper wrapper1 = new SimpleWrapper();
    wrapper1.setName("Primitive");
    wrapper1.setServletClass("PrimitiveServlet");
    Wrapper wrapper2 = new SimpleWrapper();
    wrapper2.setName("Modern");
    wrapper2.setServletClass("ModernServlet");

    Context context = new SimpleContext();
    context.addChild(wrapper1);
    context.addChild(wrapper2);

    Valve valve1 = new HeaderLoggerValve();
    Valve valve2 = new ClientIPLoggerValve();

    ((Pipeline) context).addValve(valve1);
    ((Pipeline) context).addValve(valve2);

    Mapper mapper = new SimpleContextMapper();
    mapper.setProtocol("http");
    context.addMapper(mapper);
    Loader loader = new SimpleLoader();
    context.setLoader(loader);
    // context.addServletMapping(pattern, name);
    context.addServletMapping("/Primitive", "Primitive");
    context.addServletMapping("/Modern", "Modern");
    connector.setContainer(context);
    try {
      connector.initialize();
      connector.start();

      // make the application wait until we press a key.
      System.in.read();
    }
    catch (Exception e) {
      e.printStackTrace();
    }
  }
}123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657
```

main()方法由实例化Tomcat默认连接器和两个wrappers，wrapper1和wrapper2开始。 这些包装器被命名为Primitive和Modern。 Primitive和Modern的servlet类是PrimitiveServlet和ModernServlet。

```
HttpConnector connector = new HttpConnector();
Wrapper wrapper1 = new SimpleWrapper();
wrapper1.setName("Primitive");
wrapper1.setServletClass("PrimitiveServlet");
Wrapper wrapper2 = new SimpleWrapper();
wrapper2.setName("Modern");
wrapper2.setServletClass("ModernServlet");1234567
```

然后，main()方法创建一个SimpleContext实例，并将wrapper1和wrapper2添加为SimpleContext的子容器。它还实例化两个阀门ClientIPLoggerValve和HeaderLoggerValve，并将它们添加到SimpleContext。

```
Context context = new SimpleContext();
context.addChild(wrapper1);
context.addChild(wrapper2);
Valve valve1 = new HeaderLoggerValve();
Valve valve2 = new ClientIPLoggerValve();
((Pipeline) context).addValve(valve1);
((Pipeline) context).addValve(valve2);1234567
```

接下来，它从SimpleMapper类构造一个映射器对象并将其添加到SimpleContext。 此映射器负责在上下文中查找子容器来处理HTTP请求。

```
Mapper mapper = new SimpleContextMapper();
mapper.setProtocol("http");
context.addMapper(mapper);123
```

要加载servlet类，则需要一个加载器。 在这里我们使用SimpleLoader类，正如在第一个应用Demo一样。 但是，不是将它添加到两个包装器，而是loader被添加到context中。 包装器将使用它的getLoader()找到加载器，因为context是其父级。

```
Loader loader = new SimpleLoader();
context.setLoader(loader);12
```

现在，是时候添加servlet映射了。 我们为2个包装器添加了2个匹配模式。

```
// context.addServletMapping(pattern, name);
context.addServletMapping("/Primitive", "Primitive");
context.addServletMapping("/Modern", "Modern");123
```

最后，将上下文指定为连接器的容器，并初始化和启动连接器。

```
connector.setContainer(context);
try {
    connector.initialize();
    connector.start();1234
```

**5.6.5 运行Demo**

在 Linux 下，使用冒号分开两个库：

```
java -classpath ./lib/servlet.jar:./ ex05.pyrmont.startup.Bootstrap21
```

调用PrimitiveServlet，可以使用下面的 URL 来请求：

```
http://localhost:8080/Primitive
```

调用ModernServlet，可以使用下面的 URL 来请求：

```
http://localhost:8080/Modern
```

## **5.7小结**

容器是连接器之后的第二个主模块。 容器使用许多其它模块，如Loader，Logger，Manager等。有4种类型容器：Engine，Host，Context和Wrapper。 Catalina部署没有必需所有4个容器都存在。 本章中的两个应用Demo展现了：部署可以具有单个Wrapper或含有几个wrapper的Context。