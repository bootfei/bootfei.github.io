---
title: 'spring-cloud-4-Hystrix服务熔断与服务降级'
date: 2020-12-12 15:40:07
tags:
---



> 熔断机制是服务雪崩的一种有效解决方案。常见的熔断有两种: 
>
> - 预熔断
> -  即时熔断
>
> 服务降级是请求发生问题后的一种增强用户体验的方式。
>
> - 发生服务熔断，一定会发生服务降级。
> - 但发生服务降级，并不意味着一定是发生了服务熔断。

Spring Cloud 是通过 Hystrix 来实现服务熔断与降级的。

> Hystrix 是一种开关装置，类似于熔断保险丝。在消费者端安装一个 Hystrix 熔断器，当 Hystrix 监控到某个服务发生故障后熔断器会开启，将此服务访问链路断开。不过 Hystrix 并不会将该服务的消费者阻塞，或向消费者抛出异常，而是向消费者返回一个符合预期的备选 响应(FallBack)。通过 Hystrix 的熔断与降级功能，避免了服务雪崩的发生，同时也考虑到了 用户体验。故 Hystrix 是系统的一种防御机制。

Hystrix 对于服务降级的实现方式有两种:fallbackMethod 服务降级，与 fallbackFactory服务降级。

# fallbackMethod服务降级

> 总步骤：都是针对consumer
>
> - 添加Hystrix依赖
> - 修改处理器方法。在处理器方法上添加@HystrixCommond注解
> - 在处理器中定义服务降级方法
> - 在启动类上添加@EnableCircuitBreaker注解(或将@SpringBootApplication注解替@SpringCloudApplication 注解)

- 依赖

  - ```xml
    <!--hystrix 依赖--> 
    <dependency>
    	<groupId>org.springframework.cloud</groupId> 
      <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
    ```

  - 说明 hystrix 本身与 feign 是没有关系的。

- 修改处理器方法

  - ```
    
    ```

    

# fallbackFactory服务降级