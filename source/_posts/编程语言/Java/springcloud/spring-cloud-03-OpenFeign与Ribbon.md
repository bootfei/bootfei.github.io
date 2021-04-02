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

- Copy提供者工程 **8081**，to **03-provider-8082**，**03-provider-8083**

- 更改配合文件

  - 端口号必须改变

  - <font color="red">微服务名称不能改!!！</font>

  - ```yaml
    server:
      port: 8081 #改！
    
    spring:
    	# 指定当前微服务名称
      application:
        name: abcmsc-provider-depart #不能改!!!
    ```

- 为了验证负载均衡有效，修改provider的controller返回值，加上provider的唯一标识

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

    

# Ribbon工作原理

### ILoadBalance 负载均衡器

ribbon是一个为客户端提供负载均衡功能的服务，它内部提供了一个叫做ILoadBalance的接口代表负载均衡器的操作，比如有添加服务器操作、选择服务器操作、获取所有的服务器列表、获取可用的服务器列表等等。

ILoadBalance的继承关系如下：

![Image](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhoejcjMf928HTQuV66ZKynjvROr20CAwaIlF6OydDt3vWUF5fEhRkDjztiazZmnJhOs0e2bIvrqjUw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

负载均衡器是从EurekaClient（EurekaClient的实现类为DiscoveryClient）获取服务信息，根据IRule去路由，并且根据IPing判断服务的可用性。

负载均衡器多久一次去获取一次从Eureka Client获取注册信息呢？在BaseLoadBalancer类下，BaseLoadBalancer的构造函数，该构造函数开启了一个PingTask任务setupPingTask();，代码如下：

```
    public BaseLoadBalancer(String name, IRule rule, LoadBalancerStats stats,
            IPing ping, IPingStrategy pingStrategy) {
        if (logger.isDebugEnabled()) {
            logger.debug("LoadBalancer:  initialized");
        }
        this.name = name;
        this.ping = ping;
        this.pingStrategy = pingStrategy;
        setRule(rule);
        setupPingTask();
        lbStats = stats;
        init();
    }
```

setupPingTask()的具体代码逻辑，它开启了ShutdownEnabledTimer执行PingTask任务，在默认情况下pingIntervalSeconds为10，即每10秒钟，向EurekaClient发送一次”ping”。推荐：[Java面试练题宝典](https://mp.weixin.qq.com/s?__biz=MzIyNDU2ODA4OQ==&mid=2247486846&idx=1&sn=75bd4dcdbaad73191a1e4dbf691c118a&scene=21#wechat_redirect)

```
void setupPingTask() {
        if (canSkipPing()) {
            return;
        }
        if (lbTimer != null) {
            lbTimer.cancel();
        }
        lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + name,
                true);
        lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000);
        forceQuickPing();
    }
```

PingTask源码，即new一个Pinger对象，并执行runPinger()方法。

查看Pinger的runPinger()方法，最终根据 pingerStrategy.pingServers(ping, allServers)来获取服务的可用性，如果该返回结果，如之前相同，则不去向EurekaClient获取注册列表，如果不同则通知ServerStatusChangeListener或者changeListeners发生了改变，进行更新或者重新拉取。

**完整过程是：**

LoadBalancerClient（RibbonLoadBalancerClient是实现类）在初始化的时候（execute方法），会通过ILoadBalance（BaseLoadBalancer是实现类）向Eureka注册中心获取服务注册列表，并且每10s一次向EurekaClient发送“ping”，来判断服务的可用性，如果服务的可用性发生了改变或者服务数量和之前的不一致，则从注册中心更新或者重新拉取。LoadBalancerClient有了这些服务注册列表，就可以根据具体的IRule来进行负载均衡。

### IRule 路由

IRule接口代表负载均衡策略：

```
public interface IRule{
    public Server choose(Object key);
    public void setLoadBalancer(ILoadBalancer lb);
    public ILoadBalancer getLoadBalancer();    
}
```

IRule接口的实现类有以下几种：

![Image](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhoejcjMf928HTQuV66ZKynjNJYRSuCAy0WiaP8JEeIxtHQ2q7Ss7pbArUibIwMhcJMrBumqxVuWON9g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![Image](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhoejcjMf928HTQuV66ZKynjiagxZ2sFxP5m0JBhLJXJLdNY8hia1FtLSz7iaFLR7jqdm8RQSiamPHl2zA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

其中RandomRule表示随机策略、RoundRobinRule表示轮询策略、WeightedResponseTimeRule表示加权策略、BestAvailableRule表示请求数最少策略等等。推荐：[Java面试练题宝典](https://mp.weixin.qq.com/s?__biz=MzIyNDU2ODA4OQ==&mid=2247486846&idx=1&sn=75bd4dcdbaad73191a1e4dbf691c118a&scene=21#wechat_redirect)

随机策略很简单，就是从服务器中随机选择一个服务器，RandomRule的实现代码如下：

```
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        return null;
    }
    Server server = null;
 
    while (server == null) {
        if (Thread.interrupted()) {
            return null;
        }
        List<Server> upList = lb.getReachableServers();
        List<Server> allList = lb.getAllServers();
        int serverCount = allList.size();
        if (serverCount == 0) {
            return null;
        }
        int index = rand.nextInt(serverCount); // 使用jdk内部的Random类随机获取索引值index
        server = upList.get(index); // 得到服务器实例
 
        if (server == null) {
            Thread.yield();
            continue;
        }
 
        if (server.isAlive()) {
            return (server);
        }
 
        server = null;
        Thread.yield();
    }
    return server;
}
```

RoundRobinRule轮询策略表示每次都取下一个服务器，比如一共有5台服务器，第1次取第1台，第2次取第2台，第3次取第3台，以此类推：

```
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }
 
        Server server = null;
        int count = 0;
        while (server == null && count++ < 10) {
            List<Server> reachableServers = lb.getReachableServers();
            List<Server> allServers = lb.getAllServers();
            int upCount = reachableServers.size();
            int serverCount = allServers.size();
 
            if ((upCount == 0) || (serverCount == 0)) {
                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }
 
            int nextServerIndex = incrementAndGetModulo(serverCount);
            server = allServers.get(nextServerIndex);
 
            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }
 
            if (server.isAlive() && (server.isReadyToServe())) {
                return (server);
            }
 
            // Next.
            server = null;
        }
 
        if (count >= 10) {
            log.warn("No available alive servers after 10 tries from load balancer: "
                    + lb);
        }
        return server;
    }
 
    /**
     * Inspired by the implementation of {@link AtomicInteger#incrementAndGet()}.
     *
     * @param modulo The modulo to bound the value of the counter.
     * @return The next value.
     */
    private int incrementAndGetModulo(int modulo) {
        for (;;) {
            int current = nextServerCyclicCounter.get();
            int next = (current + 1) % modulo;
            if (nextServerCyclicCounter.compareAndSet(current, next))
                return next;
        }
    }
```

WeightedResponseTimeRule继承了RoundRobinRule，开始的时候还没有权重列表，采用父类的轮询方式，有一个默认每30秒更新一次权重列表的定时任务，该定时任务会根据实例的响应时间来更新权重列表，choose方法做的事情就是，用一个(0,1)的随机double数乘以最大的权重得到randomWeight，然后遍历权重列表，找出第一个比randomWeight大的实例下标，然后返回该实例，代码略。

BestAvailableRule策略用来选取最少并发量请求的服务器：

```
public Server choose(Object key) {
    if (loadBalancerStats == null) {
        return super.choose(key);
    }
    List<Server> serverList = getLoadBalancer().getAllServers(); // 获取所有的服务器列表
    int minimalConcurrentConnections = Integer.MAX_VALUE;
    long currentTime = System.currentTimeMillis();
    Server chosen = null;
    for (Server server: serverList) { // 遍历每个服务器
        ServerStats serverStats = loadBalancerStats.getSingleServerStat(server); // 获取各个服务器的状态
        if (!serverStats.isCircuitBreakerTripped(currentTime)) { // 没有触发断路器的话继续执行
            int concurrentConnections = serverStats.getActiveRequestsCount(currentTime); // 获取当前服务器的请求个数
            if (concurrentConnections < minimalConcurrentConnections) { // 比较各个服务器之间的请求数，然后选取请求数最少的服务器并放到chosen变量中
                minimalConcurrentConnections = concurrentConnections;
                chosen = server;
            }
        }
    }
    if (chosen == null) { // 如果没有选上，调用父类ClientConfigEnabledRoundRobinRule的choose方法，也就是使用RoundRobinRule轮询的方式进行负载均衡        
        return super.choose(key);
    } else {
        return chosen;
    }
}
```

使用Ribbon提供的负载均衡策略很简单，只需以下几部：

**1、创建具有负载均衡功能的RestTemplate实例**

```
@Bean
@LoadBalanced
RestTemplate restTemplate() {
    return new RestTemplate();
}
```

使用RestTemplate进行rest操作的时候，会自动使用负载均衡策略，它内部会在RestTemplate中加入LoadBalancerInterceptor这个[拦截器]()，这个拦截器的作用就是使用负载均衡。

默认情况下会采用轮询策略，如果希望采用其它策略，则指定IRule实现，如：

```
@Bean
public IRule ribbonRule() {
    return new BestAvailableRule();
}
```

这种方式对Feign也有效。

我们也可以参考ribbon，自己写一个负载均衡实现类。

可以通过下面方法获取负载均衡策略最终选择了哪个服务实例：