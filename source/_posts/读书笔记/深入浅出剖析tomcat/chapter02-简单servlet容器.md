---
title: chapter01-简单web服务器
date: 2021-04-17 22:17:44
tags:
---



## [javax.servlet.Servlet接口](http://sishuok.com/forum/blogPost/list/4067.html;jsessionid=F387500832428F51B04251FC9481DB31)

Servlet接口需要实现下面的5个方法：

> l     public void init(ServletConfig config) throws ServletException
>
> l     public void service(ServletRequest request, ServletResponse response) throws ServletException, java.io.IOException
>
> l     public void destroy()
>
> l     public ServletConfig getServletConfig()
>
> l     public java.lang.String getServletInfo()

在某个servlet类被实例化之后，init方法由servlet容器调用。servlet容器只调用该方法一次，调用后则可以执行服务方法了。在servlet接收任何请求之前，必须是经过正确初始化的。

当一个客户端请求到达后，servlet容器就调用相应的servlet的service方法，并将Request和Response对象作为参数传入。在servlet实例的生命周期内，service方法会被多次调用。<!--这个客户请求就是为了请求这个servlet资源-->

在将servlet实例从服务中移除前，会调用servlet实例的destroy方法。一般情况下，在服务器关闭前，会发生上述情况，servlet容器会释放内存。只有当servlet实例的service方法中所有的线程都退出或执行超时后，才会调用destroy方法。当容器调用了destroy方法，就不会再调用service方法了。

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

<font color="red">注意，在这个servlet中，每次请求servlet都会载入servlet类。</font>

该程序包括6个类：HttpServer1、Request、Response、StaticResourceProcessor、ServletProcessor1、Constants。

![image](https://yqfile.alicdn.com/2d4ea4f13960600f97c3391b641a0afa78a01d15.png)



​     该程序的入口点（静态main方法）在类HttpServer1中。main方法中创建HttpServer1的实例，饭后调用其await方法。await方法等待HTTP请求，为接收到的每个请求创建request和response对象，将它们分发到一个StaticResourceProcessor类或ServletProcessor类的实例。

### HttpServer1类

应用程序1中的HttpServer1类与第1章中简单Web服务器应用程序中的HttpServer类似。但是，[该应用程序中的HttpServer1类既可以对静态资源请求，也可以对于servlet资源请求]()。

若要请求一个静态资源，可以在浏览器的地址栏或URL框中输入如下格式的URL：这与第1章的Web服务器应用程序中对静态资源的请求相同。

```
http://machineName:port/staticResource
```


若要请求servlet资源，可以使用如下格式的URL：

```
http://machineName:port/servlet/servletClass
```

但是，若要调用其他的servlet（如ModernServlet），则servlet容器抛出异常。在后面的章节中，你将学会如何构建可以兼具两种功能的servlet容器。
HttpServer1类的定义在代码清单2-2中。

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

该类实现了javax.servlet.ServletRequest接口，但并不返回实际内容。

### Response类

实现了javax.servlet.ServletResponse接口，大部分方法都返回一个空值，除了getWriter方法以外。

​     在getWriter方法中，PrintWriter类的构造函数的第二个参数表示是否启用autoFlush。因此，若是设置为false，则如果是servlet的service方法的最后一行调用打印方法，则该打印内容不会被发送到客户端。这个bug会在后续的版本中修改。

### StaticResourceProcessor类

该类用于处理对静态资源的请求。

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

 

该类很简单，只有一个process方法。

- 为了载入servlet类，需要从URI中获取servlet的类名称，由request负责；

- 为了载入servlet类，载入servlet时使用的是UrlClassLoader类，它是ClassLoader类的直接子类，有三种构造方法。
  - public URLClassLoader(URL[] urls);
  - public URL(URL context, java.lang.String spec, URLStreamHandler hander) throws MalformedURLException
  - public URL(java.lang.String protocol, java.lang.String host, java.lang.String file) throws MalformedURLException

  第一个构造函数，参数为一个Url对象的数组，每个url指明了从哪里查找servlet类。若某个Url是以“/”结尾的，则认为它是一个目录；否则，认为它是一个jar文件，必要时会将它下载并解压。 <!--这里使用的是Constans.WEB_ROOT，其实就是项目编译后的WEB_INFO/Class目录-->

- 加载并创建完servlet类以后<!--所以必须使用加载和newInstance()方式分布进行-->，将其向下转型为javax.servlet.servlet，并调用其service()方法

注：在servlet容器中，查找servlet类的位置称为repository。

在我们的应用程序中，servlet容器只需要查找一个repository，在工作目录的webroot路径下。



### 验证

Linux下启动项目

```
java -classpath ./lib/servlet.jar:./  ex02.pyrmont.HttpServer1
```

测试访问servlet

```
http://localhost:8080/servlet/Primitiveservlet
```



## Application 2

在之前的程序中，有一个严重的问题，必须将ex02.pyrmont.Request和ex02.pyrmont.Response分别转型为javax.servlet.ServletRequest和javax.servlet.ServletResponse，再作为参数传递给具体的servlet的service方法。这样并不安全，熟知servlet容器的人可以将ServletRequest和ServletResponse类向下转型为Request和Response类，并执行parse和sendStaticResource方法。

​     一种解决方案是将这两个方法的访问修饰符改为默认的（即，default），这样就可以避免包外访问。另一种更好的方案是使用外观设计模式。uml图如下：

![img](http://sishuok.com/forum/upload/2012/4/10/8cdb2ac0da1fdb61ed7cdfdb2694b664__%E6%9C%AA%E5%91%BD%E5%90%8D.jpg)



在第二个应用程序中，添加了两个façade类，RequestFacade和ResponseFacade。RequestFacade类实现了ServletRequest接口，通过在其构造方法中传入一个ServletRequest类型引用的Request对象来实例化。ServletRequest接口中每个方法的实现都会调用Request对象的相应方法。但是，ServletRequest对象本身是private类型，这样就不能从类的外部进行访问。这里也不再将Request对象向上转型为ServletRequest对象，而是创建一个RequestFacade对象，并把它传给service方法。这样，就算是将在servlet中获取了ServletRequest对象，并向下转型为RequestFacade对象，也不能再访问ServletRequest接口中的方法了，就可以避免前面所说的安全问题。

​     RequestFacade.java代码如下：

![img](http://sishuok.com/forum/upload/2012/4/10/81a96e844958fa8c316c88141885b4cd__%E6%9C%AA%E5%91%BD%E5%90%8D.jpg)

注意它的构造函数，接收一个Request对象，然后向上转型为ServletRequest对象，赋给其private成员变量request。该类的其他方法中，都是调用request的相应方法实现的，这样就将ServletRequest完整的封装得RequestFacade中了。

​     同理，ResponseFacade类也是这样的。

​     Application 2中的类包括，HttpServer2、Request、Response、StaticResourceProcessor、ServletProcessor2、Constants。