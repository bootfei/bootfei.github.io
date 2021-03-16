---
title: spring cloud 02 服务中心Eureka
date: 2020-12-12 17:08:01
tags:
---



# **Eureka** 概述

## **CAP** 定理

CAP 定理指的是在一个分布式系统中，Consistency(一致性)、 Availability(可用性)、 Partition tolerance(分区容错性)，三者不可兼得。CAP 定理的内容是:对于分布式系统，网络环境相对是不可控的，出现网络分区是不可 避免的，因此系统必须具备分区容错性。但系统不能同时保证一致性与可用性。即要么 CP， 要么 AP。

- 一致性(C):分布式系统中多个主机之间是否能够保持数据一致的特性。即，当系统数据发生更新操作后，各个主机中的数据仍然处于一致的状态。
- 可用性(A):系统提供的服务必须一直处于可用的状态，即对于用户的每一个请求，系统总是可以在有限的时间内对用户做出响应。
- 分区容错性(P):分布式系统在遇到任何网络分区故障时，仍能够保证对外提供满足一致性和可用性的服务。

## **Eureka** 简介

Eureka 就是一个专门用于服务发现的服务器，一些服务注册到该服务器，而另一 些服务通过该服务器查找其所要调用执行的服务。可以充当服务发现服务器的组件很多，例 如 Zookeeper、Consul、Eureka 等。

Eureka 与 Zookeeper 都可以充当服务中心，那么它们有什么区别呢?它们的区别主要体 现在对于 CAP 原则的支持的不同。

- Eureka:AP
- zk:CP



# 创建 **Eureka** 服务中心

> 总步骤：
>
> - 添加 Eureka Server 依赖
> - 在配置文件中配置 Eureka Server
> - 在启动类上添加@EnableEurekaServer 注解，启动 Eureka Server 功能

- Maven依赖


```xml
//eureka server依赖
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>
</dependencies>
```

- 配置文件

```yml
server:
	port: 8000
	
eureka:
  instance:
    hostname: localhost # 应用的主机名称,我一般喜欢用spring.application.name
  client:
    registerWithEureka: false #值为`false`意味着自身仅作为服务器，不作为客户端
    fetchRegistry: false #值为`false`意味着无需注册自身
    serviceUrl:
      defaultZone: http://localhost:8000/eureka/  #指明了应用的URL
      
  server:
	  #开启自我保护机制
  	enable-self-preservation: true
  	#指定开启阈值
  	renewal-percent-threshold: 0.75
```

- 定义spring boot启动类


```java
@SpringBootApplication
@EnableEurekaServer //非常关键！！！
public class Application {
}
```

http://localhost:8000/ 访问Eureka启动UI界面

# 创建provider module

注意：与实验一大致相同，只记录不同部分

> 总步骤
>
> - 添加 Eureka Client 依赖
> - 在配置文件中指定要注册的 Eureka Server 地址，指定自己微服务名称

- Maven依赖


```xml
<dependencies>
  <!--eureka 客户端依赖-->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
 
  <!--actuator依赖-->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  
</dependencies>
```

- 配置文件


```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8000/eureka/
      #若使用Eureka集群，则改为以下配置
      defaultZone: http://localhost:8000/eureka/,http://localhost:8100/eureka/
    instance:
    	instance-id: abc-msc-provider-8081

spring:
	application:
		name: abcmsc-provider-depart  #指定当前微服务对外暴露时的名称
```

- 修改启动类
  - 从官网介绍看到，provider和consumer启动类不需要做修改，eureka已经做了
  - <img src="/Users/qifei/Library/Application Support/typora-user-images/image-20201212173428818.png" alt="image-20201212173428818" style="zoom:25%;" />



# 创建consumer module

> 总步骤 
>
> - 添加 Eureka Client 依赖
> - 在配置文件中指定要注册的 Eureka Server 地址，指定自己微服务名称
> - 在JavaConfig类中为RestTemplate添加@LoadBalance注解，实例负载均衡 
> - 修改处理器，将“主机名:端口” -> “提供者微服务名称”

- 添加依赖

  - ```xml
    <dependencies>
      <!--eureka 客户端依赖-->
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
      </dependency>
     
    
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
      </dependency>
      
    </dependencies>
    ```
  
- 修改配置文件

  - ```yaml
    eureka:
    	#指定Eureka服务中心
      client:
        serviceUrl:
          defaultZone: http://localhost:8000/eureka/
          #若使用Eureka集群，则改为以下配置
          defaultZone: http://localhost:8000/eureka/,http://localhost:8100/eureka/
    
    spring:
    	application:
    		name: abcmsc-consumer-depart  #指定当前微服务对外暴露时的名称
    ```

- 修改处理器

  - ```java
    @RestController
    @RequestMapping("/consumer/depart")
    public class ConsumerController {
        @Autowired
        private RestTemplate restTemplate;
    
      	//将原来的主机：端口，改为微服务的服务名称
        private final String  SERVICE_PROVIDER="http://abcmsc-provider-depart";
    		...
    }
    ```

    

- 修改 **JavaConfig** 类

  - 修改RestTemplate

  - ```java
    @Configuration
    public class DepartCodeConfig {
      	@LoadBalanced //提供消费者负载均衡的功能
        @Bean
        public RestTemplate restTemplate(){
            return new RestTemplate();
        }
    }
    ```

# 服务发现

- 修改controller类

  - 使用org.springframework.cloud.client.discovery.DiscoveryClient类

  - ```java
    package com.abc.controller;
    
    import com.abc.bean.Depart;
    import com.abc.service.DepartService;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.cloud.client.ServiceInstance;
    import org.springframework.cloud.client.discovery.DiscoveryClient;
    import org.springframework.web.bind.annotation.*;
    
    @RestController
    @RequestMapping("/provider/depart")
    public class DepartController {
        @Autowired
        private DepartService service;
        // 声明服务发现客户端
        @Autowired
        private DiscoveryClient client;
    
        @GetMapping("/discovery")
        public List<String> discoveryHandler() {
            List<String> services = client.getServices();
            for (String name : services) {
                // 获取当前遍历微服务名称的所有提供者主机
                List<ServiceInstance> instances = client.getInstances(name);
                // 遍历所有提供者主机的详情
                for(ServiceInstance instance : instances) {
                    // 获取当前提供者的唯一标识，service id
                    String serviceId = instance.getServiceId();
                    String instanceId = instance.getInstanceId();
                    // 获取当前提供者主机的host
                    String host = instance.getHost();
                    Map<String, String> metadata = instance.getMetadata();
                    System.out.println("serviceId = " + serviceId);
                    System.out.println("instanceId = " + serviceId);
                    System.out.println("host = " + host);
                    System.out.println("metadata = " + metadata);
                }
            }
            return services;
        }
    }
    
    ```
    
    

# 服务离线

服务离线，即某服务不能对外提供服务了。服务离线的原因有两种:服务下架与服务下线。这两种方案都是基于 Actuator 监控器实现的。

> - 服务下架:将注册到 Eureka Server 中的 Eureka Client 从 Server 的注册表中移除，这样其实 Client 就无法发现该 Client 了。
> - 服务下线:Client并没有从Eureka Server的注册表中移除(其它Client仍可发现该服务)，而是通过修改服务的状态来到达其它 Client 无法调用的目的。

- 添加依赖

  - 为 Eureka Client 添加 actuator 依赖。
  - 具体pom依赖（略）

- 修改配置文件

  - ```yaml
    management:
      # 开启所有监控终端
      endpoints:
        web:
          exposure:
            include: "*"
      # 开启shutdown监控终端
      endpoint:
        shutdown:
          enabled: true
    ```

  - 发送shutdown的post请求：http://localhost:8081/actuator/shutdown

  - 发送DOWN的post请求：http://localhost:8081/actuator/service-registry   body: { 'status': 'DOWN' }

# **Eureka** 的自我保护机制

在 Eureka 服务页面中看到如下红色字体内容，表示当前 EurekaServer 启动了自我保护 机制，进入了自我保护模式。

默认情况下，EurekaServer 在 90 秒内没有检测到服务列表中的某微服务，则会自动将 该微服务从服务列表中删除。但很多情况下并不是该微服务节点(主机)出了问题，而是由 于网络抖动等原因使该微服务无法被EurekaServer发现，即无法检测到该微服务主机的心跳。 若在短暂时间内网络恢复正常，但由于 EurekaServer 的服务列表中已经没有该微服务，所以 该微服务已经无法提供服务了。

在短时间内若 EurekaServer 丢失较多微服务，即 EurekaServer 收到的心跳数量小于阈值， 为了保证系统的可用性(AP)，给那些由于网络抖动而被认为宕机的客户端“重新复活”的 机会，Eureka 会自动进入自我保护模式:服务列表只可读取、写入，不可执行删除操作。当 EurekaServer 收到的心跳数量恢复到阈值以上时，其会自动退出 Self Preservation 模式。

## 默认值修改

启动自我保护的阈值因子默认为 0.85，即 85%。即 EurekaServer 收到的心跳数量若小于 应该收到数量的 85%时，会启动自我保护机制。

自我保护机制默认是开启的，可以通过修改 EurekaServer 中配置文件来关闭。但不建议关闭。

## **GUI** 上的属性值

- Renews threshold:Eureka Server 期望每分钟收到客户端的续约总数。 count * 0.85 / 15
- Renews(lastmin):EurekaServer实际在最后一分钟收到客户端的续约数量。
- 说明:若 Renews (last min) < Renews threshold ，就会启动自我保护

# **EurekaServer** 集群

这里要搭建的 EurekaServer 集群中包含三个 EurekaServer 节点，其端口号分别为 8100、8200 与 8300。

- 创建 **00-eurekaserver-8100**

  - 修改端口号和Eureka的service-url

  - ```yaml
  eureka:
      client:
        service-url:
          # 指定当前Client所要连接的eureka Server集群
          defaultZone: http://eureka8100.com:8100/eureka,http://eureka8200.com:8200/eureka,http://eureka8300.com:8300/eureka
    ```
  
- 创建 **00-eurekaserver-8200**

- 创建 **00-eurekaserver-8300**

- 修改provider和consumer的配置文件

  - 修改端Eureka的service-url

