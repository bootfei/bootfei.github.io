---
title: chapter03-连接器
date: 2021-05-08 07:55:05
tags:
---

在简介一章里说明了，tomcat由两大模块组成：连接器（connector）和容器（container）。本章将使用连接器来增强Chapter 2的功能。

- Chapter 2使用Server创建Request和Response; 现在交给connector创建  <!--它们作为参数传递给要调用的某个的servlet的service方法-->

- Chapter 2的servlet容器仅仅能运行实现了javax.servlet.Servlet接口，并把javax.servlet.ServletRequest和javax.servlet.ServletResponse实例传递给servlet的service方法；但是由于连接器并不知道servlet的具体类型（例如，该servlet是否javax.servlet.Servlet接口，还是继承自javax.servlet.GenericServlet类，或继承自javax.servlet.http.HttpServlet类），因此连接器总是传入HttpServletRequest和HttpServletResponse的实例对象。

​     本章中所要建立的connector实际上是tomcat4中的默认连接器（将在第4章讨论）的简化版。本章中，connector和container将分离开。

  