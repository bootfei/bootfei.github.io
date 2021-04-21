---
title: chapter01-简单web服务器
date: 2021-04-17 22:17:44
tags:
---



## HTTP

### 请求

三部分 + 回车

- 请求方法	统一资源标识符(URI)	协议/版本
- 请求头
- 回车/换行: CRLF（carriage returen/LineFeed）
- 实体

### 响应

三部分

- 协议	状态码	描述
- 响应头
- 响应实体段



## Socket类

套接字是网络连接的端点。套接字使应用程序可以从网络中读取数据，可以向网络中写入数据。[不同计算机上的两个应用程序可以通过连接发送或接收字节流，以此达到相互通信的目的]()。为了从一个应用程序向另一个应用程序发送消息，需要知道另一个应用程序中套接字的IP地址和端口号。在Java中，套接字由java.net.Socket表示。
要创建一个套接字，可以使用Socket类中众多构造函数中的一个。其中一个构造函数接收两个参数：主机名和端口号。

```
public Socket (java.lang.String host, int port)
```

```
new Socket ("yahoo.com", 80);
```

一旦成功地创建了Socket类的实例，就可以使用该实例发送或接收字节流。要发送字节流，需要调用Socket类的getOutputStream()方法获取一个java.io.OutputStream对象。要发送文本到远程应用程序，通常需要使用返回的OutputStream对象创建一个java.io.PrintWriter对象。若想要从连接的另一端接收字节流，需要调用Socket类的getInputStream()方法，该法会返回一个java.io.InputStream对象。

![image](https://yqfile.alicdn.com/642c4a62ac7a1790bca190ed30d4ab8006d1fbb6.png)

![image](https://yqfile.alicdn.com/9303042d37ef3886d4aa73ce2a798cb7bae62eda.png)

## **ServerSocket类**

Socket类表示一个客户端套接字。正因如此，需要使用java.net.ServerSocket类，这是服务器套接字的实现。
ServerSocket类与Socket类并不相同。服务器套接字要等待来自客户端的连接请求。当服务器套接字收到了连接请求后，它会创建一个Socket实例来处理与客户端的通信。
要创建一个服务器套接字，可以使用ServerSocket类提供的4个构造函数中的任意一个。需要指明IP地址和服务器套接字侦听的端口号。典型情况下，IP地址可以是127.0.0.1，即服务器套接字会侦听本地机器接收到的连接请求。服务器套接字侦听的IP地址称为绑定地址。服务器套接字的另一个重要属性是backlog，后者表示在服务器拒绝接收传入的请求之前，传入的连接请求的最大队列长度。
ServerSocket类的其中一个构造函数的签名如下：

```
public ServerSocket(int port, int backLog, InetAddress bindingAddress);
```

ServerSocket对象侦听本地主机的8080端口，其backlog值为1：

```
new ServerSocket(8080, 1, InetAddress.getByName("127.0.0.1"));
```

在应用程序的入口点，也就是静态main函数中，创建一个HttpServer实例，然后调用其await()方法。顾名思义，await方法会在制定的端口上等待http请求，并对其进行处理，然后发送相应的消息回客户端。在接收到命令之前，它会一直保持等待的状态。

## HttpServer类



```none
package simpleHttpServer;

import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.InetAddress;
import java.net.ServerSocket;
import java.net.Socket;

public class HttpServer {
    
    public static final String WEB_ROOT = System.getProperty("user.dir") + File.separator
            + "webroot";
    
    private static final String SHUTDOWN_COMMAND = "/SHUTDOWN";
    
    private boolean shudown = false;
    
    public static void main(String[] args){
        HttpServer server = new HttpServer();
        server.await();
    }
    
    public void await(){
        ServerSocket serverSocket = null;
        int port = 8080;
        
        try{
            serverSocket = new ServerSocket(port,1,InetAddress.getByName("127.0.0.1"));
        }
        catch (IOException e){
            e.printStackTrace();
            System.exit(1);
        }
        
        while(!this.shudown){
            Socket socket = null;
            InputStream input = null;
            OutputStream output = null;
            
            try{
                
                socket = serverSocket.accept();
                input = socket.getInputStream();
                output = socket.getOutputStream();
                
                Request request = new Request(input);
                request.parse();
                
                Response response = new Response(output);
                response.setRequest(request);
                response.sendStaticResource();
                
                socket.close();
                
                this.shudown = request.getUri().equals(SHUTDOWN_COMMAND);
                
            }
            catch (Exception e){
                e.printStackTrace();
                continue;
            }
        }
        
    }
        
}
```


这个简单的web服务器，可以处理指定目录中的静态资源请求；用WEB_ROOT表示制定的目录

```none
public static final String WEB_ROOT = System.getProperty("user.dir") + File.separator + "webroot";
```

这里是指当前目录下的webroot文件夹下面的资源。
我们通过在游览器中输入这样的内容，进行资源的请求:
http://127.0.0.1:8080/index.html

## Request类

Request类表示一个Http请求，可以传递InputStream对象来创建Request对象，调用InputStream对象的read进行Http请求数据的读取。

```none
package simpleHttpServer;

import java.io.InputStream;

public class Request {
    private InputStream input;
    private String uri;
    
    public Request(InputStream input){
        this.input = input;
    }
    
    public void parse(){
        StringBuffer request = new StringBuffer(2048);
        
        int i;
        byte[] buffer = new byte[2048];
        try{
            i = input.read(buffer);
        }
        catch (Exception e){
            e.printStackTrace();
            i = -1;
        }
        
        for(int j=0;j<i;j++){
            request.append((char)buffer[j]);
        }
        
        System.out.print(request.toString());
        this.uri = this.parseUri(request.toString());
        
    }
    
    private String parseUri(String requestString){
        int index1,index2;
        index1 = requestString.indexOf(' ');
        if(index1 != -1){
            index2 = requestString.indexOf(' ', index1 + 1);
            if(index2 > index1){
                return requestString.substring(index1 + 1,index2 );
            }
        }
        return null;
    }
    
    public String getUri(){
        return this.uri;
    }
    
}
```



Request类最重要的两个函数是parse和ParseUri；parse()方法会调用私有方法parseUri来解析HTTP请求的uri，初次之外，并没有做太多的工作。parseuri会将解析的URI存储在变量uri中。

我们以 http://127.0.0.1:8080/index.html 请求为例，HTTP请求的请求行为
GET /index.html HTTP/1.1
parse()方法从传入的Request对象的InputStream对象中读取整个字节流，并且将字节数组存入缓冲区。然后用缓存区的数组初始化StringBuffer对象request。 这样再解析StringBuffer就可以解析到Uri。

## Response类

Response类表示Http相应。其定义如下



```none
package simpleHttpServer;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.OutputStream;

public class Response {
    
    private static final int BUFFER_SIZE = 1024;
    private Request request;
    private OutputStream output;
    
    public Response(OutputStream output){
        this.output = output;
    }
    
    public void setRequest(Request request){
        this.request = request;
    }
    
    public void sendStaticResource()throws IOException{
        
        byte[] bytes = new byte[BUFFER_SIZE];
        FileInputStream fis = null;
        
        try{
            
            File file = new File(HttpServer.WEB_ROOT,request.getUri());
            if(file.exists()){
                fis = new FileInputStream(file);
                int ch = fis.read(bytes, 0, BUFFER_SIZE);
                while(ch != -1){
                    output.write(bytes, 0, ch);
                    ch = fis.read(bytes, 0, BUFFER_SIZE);
                }
            }
            else{
                String errorMessage = "HTTP/1.1 404 File Not Found\r\n" + 
                            "Content-Type: text/html\r\n" +
                            "Content-Length:23\r\n" +
                            "\r\n" + 
                            "<h1>File Not Found</h1>";
                output.write(errorMessage.getBytes());
            }
            
        }
        catch (Exception e){
            e.printStackTrace();
        }
        finally{
            if(fis != null){
                fis.close();
            }
        }
        
    }
    
    
}
```


使用OutputStream和Request来初始化Reponse，Response比较简单，得到Request的Uri，然后读取对应的file，如果file存在，则将file中的数据读取到缓存中，并且发送给游览器；如果file不存在，那么就发送



```none
"HTTP/1.1 404 File Not Found\r\n" + 
                            "Content-Type: text/html\r\n" +
                            "Content-Length:23\r\n" +
                            "\r\n" + 
                            "<h1>File Not Found</h1>";
```



错误信息给游览器。

