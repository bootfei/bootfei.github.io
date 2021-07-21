---
title: springMVC-01-执行流程分析
date: 2021-07-21 08:34:42
tags:
---

## 官网的流程图

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

