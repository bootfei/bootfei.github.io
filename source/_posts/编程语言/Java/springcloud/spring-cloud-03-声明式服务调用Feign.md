---
title: spring-cloud-03-REST请求框架Feign
date: 2021-08-31 13:26:15
tags:
---

[推荐教程网站](https://book.itheima.net/course/1265899443273850881/1275268490151862274/1275304099226591234)

## 简介

![](https://book.itheima.net/uploads/course/images/java/5.4/image-20200623135343793.png)

Feign是Netflix开发的声明式、模板化的HTTP客户端，它可以帮助我们更快捷、优雅地调用HTTP API。Feign支持多种注解，包括Feign自带注解以及JAX-RS注解等。

当Feign与Eureka和Ribbon组合使用时，Feign就具有了负载均衡的功能。在Feign的实现下，我们只需要定义一个接口并使用注解方式配置，即可完成服务接口的绑定，不需要使用RestTemple的方式，从而简化了Ribbon自动封装服务调用客户端的开发工作量。如此看来，我们可以把Feign理解为一个Spring Cloud远程服务的框架或者工具，它能够帮助开发者用更少的代码，更好的兼容方式对远程服务进行调用。

> Feign在流程中其实与Ribbon没有任何关系，Feign封装了Http请求和RPC请求，交给Feign.client发送具体的REST请求
>
> 比如使用[OpenFeign调用下游第三方接口](http://www.yanzuoguang.com/article/740.html)



## 快速入门

### 引进依赖

Feign提供了接口，仍然需要具体的client实现，那么openFeign提供此实现

```xml
<!--feign 依赖--> 
<dependency>
	<groupId>org.springframework.cloud</groupId> 
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

### 全局配置

注意：如果是配合Eureka搭建分布式系统，那么需要配置Eureka，在此不再赘述；如果只是使用Fegin作为调用第三方接口的REST框架，那么不需要任何配置，和RestTemplate一样。

### 开启Feign

在启动类EurekaFeignClientApplication中添加@EnableFeignClients开启Feign Client功能

> 注意：如果是和Eureka配合使用，添加@EnableEurekaClient开启Eureka Client功能

```java
      import org.springframework.boot.SpringApplication;
      import org.springframework.boot.autoconfigure.SpringBootApplication;
      import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
      import org.springframework.cloud.openfeign.EnableFeignClients;
      @EnableEurekaClient
      @EnableFeignClients
      @SpringBootApplication
      public class EurekaFeignClientApplication {
          public static void main(String[] args) {
            SpringApplication.run(EurekaFeignClientApplication.class, args);
          }
      }
```



### 用法1：和Eureka搭配

实现一个简单的Feign Client <!--可以看到Feign只是接口，需要具体的实现类client-->。首先在eureka-feign-client中创建service包，并在该包下创建接口FeignService，通过添加@FeignClient注解指定要调用的[服务提供者]()。

提供者服务的url：Get http://eureka-provider/hello

```java
      import org.springframework.cloud.openfeign.FeignClient;
      import org.springframework.stereotype.Service;
      import org.springframework.web.bind.annotation.RequestMapping;
      import org.springframework.web.bind.annotation.RequestMethod;
      @Service
      @FeignClient(name = "eureka-provider")
      public interface FeignService {
          @RequestMapping(value = "/hello",method = RequestMethod.GET)
          public String sayHello();
      }
```

@FeignClient注解的name属性指定FeignService接口要调用的是eureka-provider。需要注意的是，这里name属性的值，必须是[服务提供者]()application.yml全局配置文件中指定的服务提供者的名称，而不是项目的名称。

测试：

```java
public test{
	feignService.sayHello(); //通过feing client调用服务提供者的url
}
```



### 用法2：只使用此REST框架

[OpenFeign调用下游第三方接口](http://www.yanzuoguang.com/article/740.html)

[OpenFeign请求动态url](http://www.yanzuoguang.com/article/739)