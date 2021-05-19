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

