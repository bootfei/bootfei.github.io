---
title: springMVC-01-执行流程分析
date: 2021-07-21 08:34:42
tags:
---

# 官网的流程图

![img](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/images/mvc.png)

## C: 前端控制器，DispatcherServlet

- 接收请求
- 响应结果
- 请求分发，包括获取处理类、执行处理类方法、获取处理类的处理结果

## M: 处理类/业务模型，Handler

- 处理业务，需要用户自定义
- 常见的SpringMVC提供的业务模型
  - 注解的@Controller
  - HttpRequestHandler
  - SimpleControllerHandler
  - 自定义的Bean。。。。。

## V: 视图类，view



![img](https://www.fatalerrors.org/images/blog/a432f0a05dd94c9b6d0700446c5c1e81.jpg)

## HandlerMapping和HandlerAdapter

handler的种类繁多，需要做映射：

HanlderMapping负责返回给Controller一个Handler

HandlerAdapter负责把handler转为对应的HandlerApater，然后执行



## 伪代码

DispatchServlet类

```
private HandlerMapping hm;
public void doDispatch(req, resp){
	//查找Handler, 交给HandlerMapping处理
	Object handler = hm.getHandler(req);
	
	//执行处理器方法，交给HandlerAdpater处理
	HandlerAdpater ha = getHandlerAdpater(handler);
	
	//执行方法
	ha.handlerRequest(req,resp);
}
```



# 源码分析

### DispachServlet类: 

初始化流程

- 集成了FrameworkServlet实现类，实现了init()方法，
  - init()方法调用了自身的initWebApplication()方法，
    - initWebApplication()方法调用AbstarctApplicationContext的refresh()方法，<!--和spring的高级容器关联上了-->
- 父子容器的概念
  - 

```java
#servlet被Tomat加载以后，会被调用init方法
public void init(){
	//从web.xml中读取Bean配置文件config.xml，创建SpringIOC容器
  
  //HandlerMapping类被封装为Bean, 比如BeanNameUrlHandlerMappingBean: name = urlBeanName，比如RequestHandlerMappingBean:
  
  //HandlerAdapter被封装为Bean
}
```

> C: DispatchServlet类
>
> M: @Controller标记的类+method ， 所以@Controller不是C，而是M



执行流程

- doDispatch()



### HandlerMapping类

- BeanNameUrlHandlerMapping
- RequestMappingHandlerMapping：对于使用@Controller注解和@RequestBody的注解，所以在init()方法中要扫描到这些注解对应的类
- SimpleHanlderMapping

> 在init()方法中，将config.xml中的url映射关系，交给Spring的beanFactory，所以HandlerMapping都会实现Spring的Aware()接口

```
public void init(){
	//在config.xml中配置的bean（init-methods）
}
```



### HandlerAdapter

- RequestMappingHandlerAdapter：调用RequestMappingHandlerMapping中（@Controller类）的方法



> 注意：
>
> 在底层反射实现中，compiler会把arguments编译为obj1 ... （class文件），这就会丧失意义，所以需要在pom的maven插件中，添加