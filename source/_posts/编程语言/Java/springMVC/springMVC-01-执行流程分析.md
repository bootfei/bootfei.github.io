---
title: springMVC-01-执行流程分析
date: 2021-07-21 08:34:42
tags:
---

# servlet演进到spring mvc

servlet的原理以及不足

- 开发者为每个servlet重写其doGet(HttpServletRequest，HttpServletResponse)与doPost(HttpServletRequest，HttpServletResponse)方法，**需要手动从HttpServletRequest解析出来业务参数**
- Servlet使用的是模板方法的设计模式，在Servlet顶层将会调用service方法，**开发者需要把业务逻辑重写service()方法**
- 将自定义Servlet注册到web.xml中，在web.xml中配置servlet-mapping来存储映射关系，**缺乏灵活性**
- 每个servlet只能处理一个请求，业务情景复杂的话，**需要写众多servlet**





Spring架构优化:

> https://docs.spring.io/spring-framework/docs/3.0.0.M4/spring-framework-reference/html/ch15.html#mvc-introduction
>
> The Spring Web model-view-controller (MVC) framework is designed around a `DispatcherServlet` that dispatches requests to handlers, with configurable handler mappings, view resolution, locale and theme resolution as well as support for uploading files. The default handler is based on the `@Controller` and `@RequestMapping` annotations, offering a wide range of flexible handling methods. With the introduction of Spring 3.0, the `@Controller` mechanism also allows you to create RESTful Web sites and applications, through the `@PathVarariable` annotation and other features.

- 省略众多Servlet，只用一个超级Servlet --- *DispatcherServlet*。并且把和业务无关的处理都交个它。
- 和业务有关的servlet，用业务模型进行抽象，




# 官网的流程图

![img](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/images/mvc.png)

### FrontController(抽象)，DispatcherServlet(实现)

- 接收请求
- 响应结果
- 请求分发，包括获取处理类、执行处理类方法、获取处理类的处理结果

### 业务模型Handler(抽象), Controller(实现)

- 处理业务，需要用户自定义
- 常见的SpringMVC提供的业务模型
  - 注解的@Controller
  - HttpRequestHandler
  - SimpleControllerHandler
  - 自定义的Bean。。。。。

### V: 视图类，view



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