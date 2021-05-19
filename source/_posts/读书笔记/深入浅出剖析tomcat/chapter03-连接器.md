---
title: chapter03-连接器
date: 2021-05-08 07:55:05
tags:
---

在简介一章里说明了，tomcat由两大模块组成：连接器（connector）和容器（container）。本章将使用连接器来增强Chapter 2的功能。

- Chapter 2使用Server创建Request和Response; 现在交给connector创建  <!--它们作为参数传递给要调用的某个的servlet的service方法-->
- Chapter 2的servlet容器仅仅能运行实现了javax.servlet.Servlet接口，并把javax.servlet.ServletRequest和javax.servlet.ServletResponse实例传递给servlet的service方法；但是由于连接器并不知道servlet的具体类型（例如，该servlet是否javax.servlet.Servlet接口，还是继承自javax.servlet.GenericServlet类，或继承自javax.servlet.http.HttpServlet类），因此连接器总是传入HttpServletRequest和HttpServletResponse的实例对象。

​     本章中所要建立的connector实际上是tomcat4中的默认连接器（将在第4章讨论）的简化版。本章中，connector和container将分离开。



## 3.3 Application

从本章开始，每章的应用程序都会按照模块进行划分。本章的应用程序可分为3个模块：connector、startup、core。

- 启动类模块仅包括一个启动类，负责启动应用程序。
- connector模块的类可分为以下5个部分：
  - 连接器及其支持类（HttpConnector和HttpProcessor）；
  - 表示http请求的类（HttpRequest）及其支持类；
  - 表示http响应的类（HttpResponse）及其支持类；
  - 外观装饰类（HttpRequestFacade和HttpResponseFacade）；
  - 常量类。
- core模块包括ServletProcessor类和StaticResourceProcessor类。

![img](https://www.programmersought.com/images/667/a2fa48bccff98a27f6d8df02278f4dab.png)





![img](https://www.programmersought.com/images/968/2da5e3f2b2b3ac6ff7712ad78013cc60.png)  

### 启动类（略）

### HttpConnector类

- HttpConnector类实现了java.lang.Runnable接口，为每个请求创建一个线程；

- 启动应用程序时，会创建一个HttpConnector对象，其run方法会被调用。其run方法中是一个循环体，执行以下三件事：

  l     等待http请求；

  l     为每个请求创建一个HttpPorcessor对象；

  l     调用HttpProcessor对象的process方法。

```java
public class HttpConnector implements Runnable {

		public void run(){
  		  while(){
          	...
            
            // Initialize the processor, handle the socket

            HttpProcessor httpProcessor = new HttpProcessor(this);

            httpProcessor.process(socket);
        }
    }
  
    public void start() {

        Thread thread = new Thread(this);

        thread.start();

    }

}
```



### HttpProcessor类的process(socket)方法

-  HttpProcessor类的process方法从http请求中获取socket。对每个http请求，它要做一下三件事：

  l     创建一个HttpRequest对象和一个HttpResponse对象；

  l     处理请求行（request line）和请求头（request headers），填充HttpRequest对象；

  l     将HttpRequest对象和HttpRespons

```java
public class HttpProcessor {
 
    private HttpConnector httpConnector; //通过构造函数，1对1

    private SocketInputStream inputStream; //依赖 use

    private OutputStream outputStream; //依赖 use
  
  	private HttpRequest httpRequest;
  
  	private HttpResponse httpResponse;

    private static int BYTE_SIZE = 2048;

    public HttpProcessor(HttpConnector httpConnector) {
    		this.httpConnector = httpConnector;
    }

		public void process(Socket socket) {
          try {

              inputStream = new SocketInputStream(socket.getInputStream(), BYTE_SIZE);
              outputStream = socket.getOutputStream();

               // Create an httpRequest object
              httpRequest = new HttpRequest(inputStream);

               // Create an httpResponse object
              httpResponse = new HttpResponse(outputStream);
              httpResponse.setRequest(httpRequest);
              httpResponse.setHeader("Server", "Pyromont Servlet Container");

               // Parse the Header and Request parameters
              parseRequest(input, output);
              parseHeader(input);

               / / Determine the url path, whether to call ServletResource

              if(httpRequest.getUri().contains("/servlet/")) {
                  ServletProcessor servletProcessor = new ServletProcessor();
                  servletProcessor.process(httpRequest, httpResponse);                   								// or call staticResource
              }else {
                  StaticResourceProcessor staticResourceProcessor = new StaticResourceProcessor();
                  staticResourceProcessor.process(httpRequest, httpResponse);
              }

               // close the socket
              socket.close();
          }catch (Exception e) {
	            e.printStackTrace();
          }
      }
 
}
```



