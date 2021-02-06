---
title: 'spring-cloud-03-OpenFeign与Ribbon'
date: 2020-12-12 15:40:07
tags:
---



# **OpenFeign** 简介

OpenFeign 可以将提供者提供的 Restful 服务伪装为接口进行消费，消费者只需使用“feign 接口 + 注解”的方式即可直接调用提供者提供的 Restful 服务，而无需再使用 RestTemplate。

需要注意:

-  该伪装的Feign接口是由消费者调用，与提供者没有任何关系。 
-  Feign仅是一个伪客户端，其不会对请求做任何处理。
-  Feign是通过注解的方式实现RESTful请求的。

Ribbon 是 Netflix 公司的一个开源的负载均衡 项目，是一个客户端负载均衡器，运行在消费者端。

OpenFeign 也是运行在消费者端的，使用 Ribbon 进行负载均衡，所以 OpenFeign 直接内 置了 Ribbon。即在导入 OpenFeign 依赖后，无需再专门导入 Ribbon 依赖了。



## 创建消费者工程

```
 这里无需修改提供者工程，只需修改消费者工程即可。
```

> 总步骤
>
> - 添加OpenFeign依赖
> - 定义Feign接口，指定要访问的微服务
> - 修改处理器，使用Feign接口来消费微服务
> -  将JavaConfig中的RestTemplate的创建方法删除
> - 在启动类上添加@EnableFeignClients注解

- Maven依赖


```xml
<!--feign 依赖--> 
<dependency>
	<groupId>org.springframework.cloud</groupId> 
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

- 定义 **Feign** 接口


```java
// 关于Feign的说明：
// 1)Feign接口名一般是与业务接口名相同的，但不是必须的
// 2)Feign接口中的方法名一般也是与业务接口方法名相同，但也不是必须的
// 3)Feign接口中的方法返回值类型，方法参数要求与业务接口中的相同（重要） 
// 4)接口上与方法上的Mapping的参数URI要与提供者处理器相应方法上的Mapping的URI相同（重要） 

// 指定当前为Feign客户端，参数为提供者的微服务名称
@FeignClient("abcmsc-provider-depart")
@RequestMapping("/provider/depart")
public interface DepartService {
    @PostMapping("/save")
    boolean saveDepart(@RequestBody Depart depart);

    @DeleteMapping("/del/{id}")
    boolean removeDepartById(@PathVariable("id") Integer id);

    @PutMapping("/update")
    boolean modifyDepart(@RequestBody Depart depart);

    @GetMapping("/get/{id}")
    Depart getDepartById(@PathVariable("id") Integer id);

    @GetMapping("/list")
    List<Depart> listAllDeparts();
}
```

- 删除 **JavaConfig** 类

  - ```java
    public class DepartCodeConfig{}
    ```
    
    

- 修改处理器


```java
@RestController
@RequestMapping("/provider/depart")
public class ProviderController {
		//删除RestTemplate
  	//private RestTemplate restTemplate;
  
  	//使用openFeign代理，spring自动生成代理类
    @Autowired
    private DepartService service;

    @PostMapping("/save")
    public boolean saveHandle(@RequestBody Depart depart) {
        return service.save(depart);
    }

    @GetMapping("/get/{id}")
    public Depart getHandle(@PathVariable(value = "id")int id){
        return service.getDepartById(id);
    }
}
```

- 修改启动类

  - ```java
    @EnableFeignClients  // 开启Feign客户端
    @SpringBootApplication
    public class ApplicationConsumer8080 {}
    ```



## 超时设置

Feign 连接提供者（connectTimeout）、对于提供者的调用(readTimeout)均可设置超时时限。

- 消费者

  - 修改配置文件

    ```yaml
    feign:
      client:
        config:
          default:
            connectTimeout: 5000   # 指定Feign客户端连接提供者的超时时限
            readTimeout: 5000      # 指定Feign客户端连接上提供者后，向提供者进行提交请求，从提交时刻开始，到接收到响应，这个时段的超时时限
    
    ```

    

- 提供者：为了演示效果，延长service的执行时长

  - 修改service

    ```java
    TimeUnit.SECONDS.sleep(5s)
    ```



## **Gzip** 压缩设置

Feign 支持对请求(Feign 客户端向提供者的请求, 即consumer发送给provider的请求)和响应(Feign 客户端向客户端浏览器的响应，即provider响应给浏览器的请求)进行 Gzip 压缩以提高通信效率。

​													

> 浏览器（客户端） ------>    consumer  （压缩）----->   provider		
>
> 浏览器（客户端） <------（压缩）    consumer <-----   provider

- 修改配置文件

  ```yaml
  feign:
  	compression:
  		request:
  			enable: true #开启consumer对provider请求的压缩
  			mime-type: ["text/xml","application/json"] #针对哪些mime类型的文件进行压缩
  		response: 
  			enable: true #开启consumer对客户端响应的压缩
  ```

  

# **Ribbon** 负载均衡

- 复制提供者工程 **8081**，to **03-provider-8082**，**03-provider-8083**

- 更改配合文件

  - 端口号必须改变

  - 微服务名称不能改！

  - ```yaml
    server:
      port: 8081 #改！
    
    spring:
    	# 指定当前微服务名称
      application:
        name: abcmsc-provider-depart #不能改！
    ```

    

# Ribbon 更换负载均衡策略

Ribbon 默认采用的是 RoundRobinRule，即轮询策略。但通过[修改消费者工程的配置文件]()，或修改[消费者的启动类]()或 [JavaConfig 类]()可以实现更换负载均衡策略的目的。

- 方式1：修改配置文件

  - ```yaml
    # 修改负载均衡策略
    abcmsc-provider-depart: # 要负载均衡的提供者微服务名称
    	ribbon: # 指定要使用的负载均衡策略
    		NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
    ```

- 方式2：修改 **JavaConfig** 类

  - ```java
    在 JavaConfig 类中添加负载负载 Bean 方法。
    @Bean
    public IRule getRule(){
    	return new RandomRule();
    }
    ```

# Ribbon 自定义负载均衡策略

Ribbon 支持自定义负载均衡策略。负载均衡算法类需要实现 IRule 接口。

该负载均衡策略的思路是:从所有可用的 provider 中排除掉指定端口号的 provider，剩 余 provider 进行随机选择。

需要修改JavaConfig类

- 实现IRule接口

  - ```java
    package com.abc.balance;
    
    import com.netflix.loadbalancer.ILoadBalancer;
    import com.netflix.loadbalancer.IRule;
    import com.netflix.loadbalancer.Server;
    /**
     * 从所有可用的provider中排除掉指定端口号的provider，剩余provider进行随机选择。
     */
    public class CustomRule implements IRule {
        private ILoadBalancer lb;
        // 记录所有要排除的端口号
        private List<Integer> excludePorts;
    
        public CustomRule() {
        }
    
        public CustomRule(List<Integer> excludePorts) {
            this.excludePorts = excludePorts;
        }
    
        @Override
        public Server choose(Object key) {
            // 获取所有UP状态的server
            List<Server> servers = lb.getReachableServers();
            // 获取到排除了指定端口的所有剩余Servers
            List<Server> availableServers = getAvailableServers(servers);
            // 对剩余的Servers通过随机方式获取一个Server
            return getAvailableRandomServer(availableServers);
        }
    
        // 获取到排除了指定端口的所有剩余Servers
        // 使用Lambda方式实现
        private List<Server> getAvailableServers(List<Server> servers) {
            // 若没有指定要排除的port，则直接返回所有Server
            if(excludePorts == null || excludePorts.size() == 0) {
                return servers;
            }
    
            // 用于存放真正可用的Server
            List<Server> aservers = servers.stream()  /
                        // noneMatch()：用于判断stream中的元素是否全部都不符合。只要找到一个符合的元素该方法就返回false
                        .filter(server -> excludePorts.stream().noneMatch(port -> server.getPort() == port))
                        .collect(Collectors.toList());
    
            return aservers;
        }
    
        // 对剩余的Servers通过随机方式获取一个Server
        private Server getAvailableRandomServer(List<Server> servers) {
            // 获取一个[0，servers.size())的随机整数
            int index = new Random().nextInt(servers.size());
            return servers.get(index);
        }
    
        @Override
        public void setLoadBalancer(ILoadBalancer lb) {
            this.lb = lb;
        }
    
        @Override
        public ILoadBalancer getLoadBalancer() {
            return lb;
        }
    }
    
    ```

- 修改 **JavaConfig** 类

  - ```java
    // 修改负载均衡策略为：自定义策略
        @Bean
        public IRule loadBalanceRule() {
            List<Integer> excludePorts = new ArrayList<>();
            excludePorts.add(8083);
            return new CustomRule(excludePorts);
        }
    ```

    