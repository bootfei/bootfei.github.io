---
title: chapter02-简单的Servlet容器
date: 2021-04-17 22:17:44
tags:
---

<!--servlet是什么？提供了什么功能？在整个web框架中处于什么位置？-->

## [javax.servlet.Servlet接口](http://sishuok.com/forum/blogPost/list/4067.html;jsessionid=F387500832428F51B04251FC9481DB31)

Servlet接口需要实现下面的5个方法：

```java
//Servlet 5个方法里，其中init()、service()、destroy()3个方法是servlet的生命周期方法。当servlet类被装载初始化后，servlet容器调用init()方法。servlet 容器只调用一次，以此表明servlet 已经被加载进服务中（The servlet container calls this method exactly once to indicate to the servlet that the servlet is being placed into service）。


//在servlet收到任何请求之前，init()必须成功执行完。一个servlet程序可以重写此方法——添加那些紧需要执行一次的初始化代码，比如：加载数据库驱动、初始值等等。另一种情况，通常此方法都空着，不写任何代码。
public void init(ServletConfig config) throws ServletException 
  
//每当对此servlet发起请求时，Servlet容器都会调用其service()方法。Servlet容器传递javax.servlet.ServletRequest和javax.servlet.ServletResponse二个对象。ServletRequest对象包含客户端HTTP请求信息，ServletResponse对象封装servlet应答信息。在Servlet生命周期中，service()方法会被多次调用。<!--这个客户请求就是为了请求这个servlet资源-->
public void service(ServletRequest request, ServletResponse response) throws ServletException, java.io.IOException 
  
 
//当从服务中移除该servlet实例之前，Servlet容器会调用其destroy()方法。这通常发生在当Servlet容器关闭或Servlet容器需要更多空闲内存时。仅仅在所有 servlet 线程的 service() 方法已经退出或者超时淘汰时，destroy()方法才被调用（This method is called only after all threads within the servlet’s service() method have exited or after a timeout period has passed）。当Servlet容器调用了destroy()后，在同一个servlet中不可再调用service()方法。destroy()方法给servlet提供了释放它当初占用资源的机会，比如：内存、文件句柄、线程，并且确保任何持久化数据状态和servlet当前内存中状态同步一致。
public void destroy() 
  
//
public ServletConfig getServletConfig() 
public java.lang.String getServletInfo()
```







##  Application 1

下面从servlet容器的角度观察servlet的开发。在一个全功能servlet容器中，对servlet的每个HTTP请求来说，容器要做下面几件事：

- 当第一次调用servlet时，要载入servlet类，调用init方法（仅此一次）；
- 针对每个request请求，创建一个Request对象和一个Resposne对象；
- 调用相应的servlet的service方法，将Request对象和Response对象作为参数传入；
- 当关闭servlet时，调用destroy方法，并卸载该servlet类。

这里建立的servlet容器是一个很小的容器，没有实现所有的功能。因此，它仅能运行非常简单的servlet类，无法调用servlet的init和destroy方法。它能执行功能如下所示：

-  等待HTTP请求；
- 创建Request和Response对象；
- [若请求的是一个静态资源，则调用StaticResourceProcessor对象的process方法，传入request和response对象;]() 
- [若请求的是servlet，则载入相应的servlet类，调用service方法，传入request对象和response对象。]()<!--其实servlet就是用户请求的一种资源-->

<font color="red">注意: 在这个servlet中，每次请求servlet都会使用Class Loader载入servlet类。</font>

该程序包括6个类：HttpServer1、Request、Response、StaticResourceProcessor、ServletProcessor1、Constants。

![image](https://yqfile.alicdn.com/2d4ea4f13960600f97c3391b641a0afa78a01d15.png)



此应用Demo的入口(静态main()方法)在HttpServer1中。此main()方法创建了HttpServer1一个实例，并调用其await()方法。await()等待HTTP请求，为每次请求创建Request和Response对象，并且分发给StaticResourceProcessor实例或ServletProcessor实例——取决于请求的是静态资源还是servlet。

其中Constants类定义了其他类引用了的常量WEB_ROOT。WEB_ROOT标明了可被此Servlet容器使用的PrimitiveServlet和静态资源的位置地址。

HttpServer1实例保持等待接收HTTP请求直到接收到shutdown命令。发起shutdow命令和你第一章中操作一样。

### HttpServer1类

本应用Demo中的HttpServer1类似于第一章中的HttpServer类。然而，本HttpServer1可以服务于静态资源和servlet。当请求静态资源时，可在你的浏览器上输入如下类似URL：

[http://machineName:port/staticResource](http://machinename:port/staticResource)

就像是在第 1 章提到的，你可以请求一个静态资源。

请求一个servlet时，使用如下类似URL：

[http://machineName:port/servlet/servletClass](http://machinename:port/servlet/servletClass)

因此，假如你在本地请求一个名为 PrimitiveServlet 的 servlet，你在浏览器的地址栏或
者网址框中敲入：

http://localhost:8080/servlet/PrimitiveServlet

本Servlet容器可以服务于PrimitiveServlet。如果你调用其他的servlet，如ModernServlet，那么此Servlet容器将会抛出异常。在后面的章节中，我们将会改造此应用，使其服务于更多的servlet。

```java
public class HttpServer1 {
    /**
     * WEB_ROOT is the directory where our HTML and other files reside.
     * For this package, WEB_ROOT is the "webroot" directory under the working
     * directory.
     * The working directory is the location in the file system
     * from where the java command was invoked.
     */
    // shutdown command
    private static final String SHUTDOWN_COMMAND = "/SHUTDOWN";

    // the shutdown command received
    private boolean shutdown = false;

    public static void main(String[] args) {
        HttpServer1 server = new HttpServer1();
        server.await();
    }

    public void await() {
        ServerSocket serverSocket = null;
        int port = 8080;
        try {
            serverSocket = new ServerSocket(port, 1, InetAddress.getByName("127.0.0.1"));
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }

        // Loop waiting for a request
        while (!shutdown) {
            Socket socket = null;
            InputStream input = null;
            OutputStream output = null;
            try {
                socket = serverSocket.accept();
                input = socket.getInputStream();
                output = socket.getOutputStream();

                // create Request object and parse
                Request request = new Request(input);
                request.parse();

                // create Response object
                Response response = new Response(output);
                response.setRequest(request);

                // check if this is a request for a servlet or a static resource              
                if (request.getUri().startsWith("/servlet/")) { // a request for a servlet begins with "/servlet/"
                    ServletProcessor1 processor = new ServletProcessor1();
                    processor.process(request, response);
                } else {
                    StaticResourceProcessor processor = new StaticResourceProcessor();
                    processor.process(request, response);
                }

                // Close the socket
                socket.close();
                //check if the previous URI is a shutdown command
                shutdown = request.getUri().equals(SHUTDOWN_COMMAND);
            } catch (Exception e) {
                e.printStackTrace();
                System.exit(1);
            }
        }
    }
}
```

该类的await()方法会一直等待HTTP请求，直到接收到一条关闭命令，这点与第1章中的await()方法类似。区别在于，本章中的await()方法可以将HTTP请求分发给StaticResourceProcessor对象或ServletProcessor对象来处理。[当URI包含字符串“/servlet/”时，会把请求转发给servletProcessor对象处理]()。 <!--这就是UML类图中HttpServer持有1个ServletProcessor的原因--> 否则的话，把HTTP请求传递给StaticResourceProcessor对象处理。

### Request类

一个servlet的service()方法从servlet容器中接收javax.servlet.ServletRequest和javax.servlet.ServletResponse实例。[也就是说，对于每一个HTTP请求，servlet容器必须创建ServletRequest和ServletResponse对象，并且把它们传递给servlet的service()方法。]()

本Request类代表着一个请求对象被传递给servlet的service()方法。照此，它必须实现javax.servlet.ServletRequest 接口。本类实现了接口提供的所有方法。不过，我们想要让它非常简单，所以仅仅提供实现其中一些方法，我们在以下各章中再实现全部的方法。要编译此Request 类，你需要把这些方法的实现留空。查看本Request 类，你将会看到那些需要返回一个对象的方法返回了 null。

Request类代码如下：

```java
package org.how.tomcat.works.ex02;

import java.io.InputStream;
import java.io.IOException;
import java.io.BufferedReader;
import java.io.UnsupportedEncodingException;
import java.util.Enumeration;
import java.util.Locale;
import java.util.Map;
import javax.servlet.RequestDispatcher;
import javax.servlet.ServletInputStream;
import javax.servlet.ServletRequest;


public class Request implements ServletRequest {

  private InputStream input;
  private String uri;

  public Request(InputStream input) {
    this.input = input;
  }

  public String getUri() {
    return uri;
  }

  private String parseUri(String requestString) {
    int index1, index2;
    index1 = requestString.indexOf(' ');
    if (index1 != -1) {
      index2 = requestString.indexOf(' ', index1 + 1);
      if (index2 > index1)
        return requestString.substring(index1 + 1, index2);
    }
    return null;
  }

  public void parse() {
    // Read a set of characters from the socket
    StringBuffer request = new StringBuffer(2048);
    int i;
    byte[] buffer = new byte[2048];
    try {
      i = input.read(buffer);
    }
    catch (IOException e) {
      e.printStackTrace();
      i = -1;
    }
    for (int j=0; j<i; j++) {
      request.append((char) buffer[j]);
    }
    System.out.print(request.toString());
    uri = parseUri(request.toString());
  }

  /* implementation of the ServletRequest*/
  public Object getAttribute(String attribute) {
    return null;
  }

  public Enumeration getAttributeNames() {
    return null;
  }

  public String getRealPath(String path) {
    return null;
  }

  public RequestDispatcher getRequestDispatcher(String path) {
    return null;
  }

  public boolean isSecure() {
    return false;
  }

  public String getCharacterEncoding() {
    return null;
  }

  public int getContentLength() {
    return 0;
  }

  public String getContentType() {
    return null;
  }

  public ServletInputStream getInputStream() throws IOException {
    return null;
  }

  public Locale getLocale() {
    return null;
  }

  public Enumeration getLocales() {
    return null;
  }

  public String getParameter(String name) {
    return null;
  }

  public Map getParameterMap() {
    return null;
  }

  public Enumeration getParameterNames() {
    return null;
  }

  public String[] getParameterValues(String parameter) {
    return null;
  }

  public String getProtocol() {
    return null;
  }

  public BufferedReader getReader() throws IOException {
    return null;
  }

  public String getRemoteAddr() {
    return null;
  }

  public String getRemoteHost() {
    return null;
  }

  public String getScheme() {
   return null;
  }

  public String getServerName() {
    return null;
  }

  public int getServerPort() {
    return 0;
  }

  public void removeAttribute(String attribute) {
  }

  public void setAttribute(String key, Object value) {
  }

  public void setCharacterEncoding(String encoding)
    throws UnsupportedEncodingException {
  }

}
```



### Response类

本Response类实现了javax.servlet.ServletResponse。同样，此类必须实现接口提供的所有方法。 类似于Request类，我们除了getWriter()方法外留白了其它暂未具体实现的方法。

Response类代码如下：

```
package org.how.tomcat.works.ex02;

import java.io.OutputStream;
import java.io.IOException;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.File;
import java.io.PrintWriter;
import java.util.Locale;
import javax.servlet.ServletResponse;
import javax.servlet.ServletOutputStream;

public class Response implements ServletResponse {

  private static final int BUFFER_SIZE = 1024;
  Request request;
  OutputStream output;
  PrintWriter writer;

  public Response(OutputStream output) {
    this.output = output;
  }

  public void setRequest(Request request) {
    this.request = request;
  }

  /* This method is used to serve a static page */
  public void sendStaticResource() throws IOException {
    byte[] bytes = new byte[BUFFER_SIZE];
    FileInputStream fis = null;
    try {
      /* request.getUri has been replaced by request.getRequestURI */
      File file = new File(Constants.WEB_ROOT, request.getUri());
      fis = new FileInputStream(file);
      /*
         HTTP Response = Status-Line
           *(( general-header | response-header | entity-header ) CRLF)
           CRLF
           [ message-body ]
         Status-Line = HTTP-Version SP Status-Code SP Reason-Phrase CRLF
      */
      int ch = fis.read(bytes, 0, BUFFER_SIZE);
      while (ch!=-1) {
        output.write(bytes, 0, ch);
        ch = fis.read(bytes, 0, BUFFER_SIZE);
      }
    }
    catch (FileNotFoundException e) {
      String errorMessage = "HTTP/1.1 404 File Not Found\r\n" +
        "Content-Type: text/html\r\n" +
        "Content-Length: 23\r\n" +
        "\r\n" +
        "<h1>File Not Found</h1>";
      output.write(errorMessage.getBytes());
    }
    finally {
      if (fis!=null)
        fis.close();
    }
  }



  /** implementation of ServletResponse  */
  public void flushBuffer() throws IOException {
  }

  public int getBufferSize() {
    return 0;
  }

  public String getCharacterEncoding() {
    return null;
  }

  public Locale getLocale() {
    return null;
  }

  public ServletOutputStream getOutputStream() throws IOException {
    return null;
  }

  public PrintWriter getWriter() throws IOException {
    // autoflush is true, println() will flush,
    // but print() will not.
    writer = new PrintWriter(output, true);
    return writer;
  }

  public boolean isCommitted() {
    return false;
  }

  public void reset() {
  }

  public void resetBuffer() {
  }

  public void setBufferSize(int size) {
  }

  public void setContentLength(int length) {
  }

  public void setContentType(String type) {
  }

  public void setLocale(Locale locale) {
  }
}
```



### StaticResourceProcessor类

本StaticResourceProcessor类服务于静态资源请求。 它只有一个process()方法。可见process()方法接收二个参数：org.how.tomcat.works.ex02.Request实例和org.how.tomcat.works.ex02.Response实例。此方法只是简单调用Response对象的sendStaticResource()。

StaticResourceProcessor类代码如下：

```
public class StaticResourceProcessor {
    public void process(Request request, Response response) {
        try {
            response.sendStaticResource();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### ServletProcessor1类

​     该类用于处理对servlet资源的请求。

```java
public class ServletProcessor1 {
    public void process(Request request, Response response) {
        String uri = request.getUri();
        String servletName = uri.substring(uri.lastIndexOf("/") + 1);
        URLClassLoader loader = null;
        try {
            URL[] urls = new URL[1];
            URLStreamHandler streamHandler = null;
            File classPath = new File(Constants.WEB_ROOT); //类加载器需要加载的目标地址

            String repository = (new URL("file", null, classPath.getCanonicalPath() + File.separator)).toString();
            urls[0] = new URL(null, repository, streamHandler);
            loader = new URLClassLoader(urls);
        } catch (Exception e ) {
            e.printStackTrace();
        }

        Class myClass = null;
        try {
            myClass = loader.loadClass(servletName);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

        Servlet servlet = null;
        try {
            servlet = (Servlet) myClass.newInstance();  //很有意思，使用的是newInstance()而不是new，因为把类加载与类实例化分开了
            servlet.service((ServletRequest) request, (ServletResponse) response);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

本ServletProcessor1还是相当的简单，它只有process()一个方法。此方法接收2个参数：javax.servlet.ServletRequest实例和javax.servlet.ServletResponse实例。该方法从ServletRequest 中通过调用 getRequestUri 方法获得 URI：

```
String uri = request.getUri();1
```

请记住 URI 是以下形式的：

/servlet/servletName

在此 servletName是servlet类的名字。

要加载 servlet 类，我们需要从 URI 中知道 servlet 的名称。我们可以使用下一行代码来获得 servlet 的名字：

```
String servletName = uri.substring(uri.lastIndexOf("/") + 1);1
```

接下去，process()方法加载 servlet。要完成这个，你需要创建一个类加载器并告诉这个类加载器要加载的类的位置。对于这个 servlet 容器，类加载器直接在 Constants.WEB_ROOT 指向的目录里边查找。Constants.WEB_ROOT就是指向工作目录下面的 webroot目录。

*注意： 类加载器将在第 8 章详细讨论。*

要加载 servlet，你可以使用 java.net.URLClassLoader 类，它是 java.lang.ClassLoader类的一个直接子类。当你拥有一个 URLClassLoader 实例，你可以使用它的loadClass()方法去加载一个 servlet 类。实例化URLClassLoader是简单的(Instantiating the URLClassLoader class is straightforward)。这个类有三个构造方法，其中最简单的是：

```
public URLClassLoader(URL[] urls);1
```

这里 urls 是一个 java.net.URL 的对象数组，这些对象指向了加载类时候要查找的位置。任何以/结尾的 URL 都假设是一个目录。否则，会假定是一个将被下载并在需要的时候打开的 JAR 文件。

注意：在一个 servlet 容器里边，一个类加载器可以找到 servlet 的地方被称为资源库(repository）。

在我们的应用Demo里边，类加载器必须查找的地方只有一个，如工作目录下面的 webroot目录。因此，我们首先创建一单个 URL 组成的数组。URL 类提供了一系列的构造方法，所以有很多种方式构造一个 URL 对象。对于这个Demo来说，我们使用了和Tomcat 中另一个类的相同的构造方法。这个构造方法如下所示：

```
public URL(URL context, java.lang.String spec, URLStreamHandler hander)throws MalformedURLException1
```

你可以使用这个构造方法，并为第二个参数传递一个值，为第一个和第三个参数都传递null。不过，这里还有另外一个接受三个参数的构造方法：

```
public URL(java.lang.String protocol, java.lang.String host,
java.lang.String file) throws MalformedURLException12
```

因此，假如你使用下面的代码时，编译器将不会知道你指的是哪个构造方法：

```
new URL(null, aString, null);1
```

你可以通过告诉编译器第三个参数的类型来避开这个问题，例如：

```
URLStreamHandler streamHandler = null;
new URL(null, aString, streamHandler);12
```

对于第二个参数，你可以使用如下面的代码组成一个包含资源库(servlet 类可以被找到的地方)的字符串：

```
String repository = (new URL("file", null,
classPath.getCanonicalPath() + File.separator)).toString() ;12
```

把所有的片段组合在一起，就是process() 方法中用来构造URLClassLoader 实例时的部分代码:

```
// create a URLClassLoader
URL[] urls = new URL[1];
URLStreamHandler streamHandler = null;
File classPath = new File(Constants.WEB_ROOT);
String repository = (new URL("file", null,
classPath.getCanonicalPath() + File.separator)).toString() ;
urls[0] = new URL(null, repository, streamHandler);
loader = new URLClassLoader(urls);12345678
```

注意：
用来生成资源库的代码是从 org.apache.catalina.startup.ClassLoaderFactory
的 createClassLoader()方 法 来 的 ， 而 生 成 URL 的 代 码 是 从
org.apache.catalina.loader.StandardClassLoader 的 addRepository()方法来的。不过，在以下各章之前你不必担心这些类。

当有了一个类加载器，你可以使用loadClass()方法加载一个 servlet：

```
Class myClass = null;
try {
    myClass = loader.loadClass(servletName);
}
catch (ClassNotFoundException e) {
    System.out.println(e.toString());
}1234567
```

然后，process()方法创建一个 servlet 类的实例, 并把它向下转换为javax.servlet.Servlet, 且调用 servlet 的 service() 方法：

```
Servlet servlet = null;
try {
    servlet = (Servlet) myClass.newInstance();
    servlet.service((ServletRequest) request,(ServletResponse)      response);
}catch (Exception e) {
    System.out.println(e.toString());
}catch (Throwable e) {
    System.out.println(e.toString());
}123456789
```

**2.2.6 运行Demo**

Windows 上运行该应用程序，在工作目录下面敲入以下命令：

```
java -classpath ./lib/servlet.jar;./ org.how.tomcat.works.ex02.HttpServer11
```

Linux 下，你使用一个冒号来分隔两个库：

```
java -classpath ./lib/servlet.jar:./ org.how.tomcat.works.ex02.HttpServer11
```

要测试该应用程序，在浏览器的地址栏或者网址框中敲入：

http://localhost:8080/index.html

或者

http://localhost:8080/servlet/PrimitiveServlet

当调用 PrimitiveServlet 时，你将会在浏览器看到下面的文本：

Hello. Roses are red.

请注意，因为只是第一个字符串被刷新到浏览器，所以你不能看到第二个字符串 “Violets are
blue”。我们将在第 3 章修复这个问题。

## Application 2

在之前的程序中，有一个严重的问题，必须将ex02.pyrmont.Request和ex02.pyrmont.Response分别转型为javax.servlet.ServletRequest和javax.servlet.ServletResponse，再作为参数传递给具体的servlet的service方法。这样并不安全，熟知servlet容器的人可以将ServletRequest和ServletResponse类向下转型为Request和Response类，并执行parse和sendStaticResource方法。

- 一种解决方案是将这两个方法的访问修饰符改为默认的（即，default），这样就可以避免包外访问。<!--很多第三方jar包都这样使用，比如shiro-->

- 另一种更好的方案是使用外观设计模式。uml图如下：

![img](http://sishuok.com/forum/upload/2012/4/10/8cdb2ac0da1fdb61ed7cdfdb2694b664__%E6%9C%AA%E5%91%BD%E5%90%8D.jpg)



在第二个应用程序中，添加了两个façade类，RequestFacade和ResponseFacade。RequestFacade类实现了ServletRequest接口，通过在其构造方法中传入一个ServletRequest类型引用的Request对象来实例化。ServletRequest接口中每个方法的实现都会调用Request对象的相应方法。但是，ServletRequest对象本身是private类型，这样就不能从类的外部进行访问。这里也不再将Request对象向上转型为ServletRequest对象，而是创建一个RequestFacade对象，并把它传给service方法。这样，就算是将在servlet中获取了ServletRequest对象，并向下转型为RequestFacade对象，也不能再访问ServletRequest接口中的方法了，就可以避免前面所说的安全问题。



> 注意: 它的构造函数，接收一个Request对象，然后向上转型为ServletRequest对象，赋给其private成员变量request。该类的其他方法中，都是调用request的相应方法实现的，这样就将ServletRequest完整的封装得RequestFacade中了。

 serveltProcess方法修改以下部分：

```java
        try {
            servlet = (Servlet) myClass.newInstance();  //很有意思，使用的是newInstance()而不是new，因为把类加载与类实例化分开了, tomcat需要破坏双亲委派，使用线程级别的类加载器
            servlet.service((ServletRequest) requestFacade, (ServletResponse) responseFacade);
        } catch (Exception e) {
            e.printStackTrace();
        }
```

