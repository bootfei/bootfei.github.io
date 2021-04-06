---
title: Ribbon的负载均衡策略及原理
date: 2021-04-02 12:50:59
tags:
---

## ILoadBalance 负载均衡器

[ribbon是一个为客户端提供负载均衡功能的服务(客户端从Eureka下载提供者列表，ribbo负责选择提供者)]()，它内部提供了一个叫做ILoadBalance的接口代表负载均衡器的操作，比如有添加服务器操作、选择服务器操作、获取所有的服务器列表、获取可用的服务器列表等等。

### ILoadBalance的继承关系

![Image](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhoejcjMf928HTQuV66ZKynjvROr20CAwaIlF6OydDt3vWUF5fEhRkDjztiazZmnJhOs0e2bIvrqjUw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

负载均衡器是从EurekaClient（EurekaClient的实现类为DiscoveryClient）获取服务信息，根据IRule去路由，并且根据IPing判断服务的可用性。

### 从Eureka获取注册信息

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

setupPingTask()的具体代码逻辑，它开启了ShutdownEnabledTimer执行PingTask任务，在默认情况下pingIntervalSeconds为10，即每10秒钟，向EurekaClient发送一次”ping”。

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

## IRule 路由

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

其中RandomRule表示随机策略、RoundRobinRule表示轮询策略、WeightedResponseTimeRule表示加权策略、BestAvailableRule表示请求数最少策略等等。

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

## 验证并使用

使用Ribbon提供的负载均衡策略很简单，只需以下几部：

**1、创建具有负载均衡功能的RestTemplate实例**

```
@Bean
@LoadBalanced
RestTemplate restTemplate() {
    return new RestTemplate();
}
```

使用RestTemplate进行rest操作的时候，会自动使用负载均衡策略，它内部会在RestTemplate中加入LoadBalancerInterceptor这个拦截器，<!--可以看到，注解的实现就是通过拦截--> 这个拦截器的作用就是使用负载均衡。

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

```
 @Autowired
 LoadBalancerClient loadBalancerClient; 
 
 //测试负载均衡最终选中哪个实例
 public String getChoosedService() {
     ServiceInstance serviceInstance = loadBalancerClient.choose("USERINFO-SERVICE");
     StringBuilder sb = new StringBuilder();
     sb.append("host: ").append(serviceInstance.getHost()).append(", ");
     sb.append("port: ").append(serviceInstance.getPort()).append(", ");
     sb.append("uri: ").append(serviceInstance.getUri());
     return sb.toString();
 }
```