---
title: 'spring-cloud-5-微服务网关Zuul'
date: 2020-12-12 15:40:07
tags:
---

# 概述

网关是系统唯一对外的入口，介于客户端(eg.浏览器)与服务器端之间，用于对请求进行鉴权、限流、 路由、监控等功能

Zuul 主要提供了对请求的路由与过滤功能。

- 路由:将外部请求转发到具体的微服务实例上，是外部访问微服务的统一入口。 
- 过滤:对请求的处理过程进行干预，对请求进行校验、鉴权等处理。

Zuul也是Eureka client，从Eureka server获取其他consumer的信息。

# 创建 **zuul** 网关工程

- 导入依赖

  - 首先除了 eureka client 依赖后，将其它依赖全部删除，然后再导入 zuul 依赖。

  - ```xml
    <!--zuul 依赖--> 
    <dependency>
    <groupId>org.springframework.cloud</groupId> 
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
    ```

- 修改启动类

  - @EnableZuulProxy

- 修改配置文件

  - 与其他Eureka client一样，向Eureka发送信息和从Eureka获取信息

  - ```yaml
    server:
    	port: 9000
    eureka:
    	client:
    		service-url:
			default-zone: http//...
    ```
    

> 如何测试：
>
> - 访问路径 = zuul地址 + [consumer微服务名称]() + consumer接口
>
> - 访问http://localhost:9000/abcmsc-consumer-depart-8080/consumer/get/1

# 路由策略配置

前面的访问方式，需要将[微服务名称]()暴露给用户，会存在安全性问题。所以，可以自定义路径来替代微服务名称，即自定义路由策略。

- 修改配置文件

  - 在配置文件中添加如下配置。
  
  - ```yaml
    zuul:
    	routes:
    		#指定路由规则
    		abcmsc-consumer-depart-8080: /abc8000/**
    		abcmsc-consumer-depart-8090: /abc9000/**
    		
    ```

> 如何测试：
>
> - 访问路径 = zuul地址 + zuul的路由规则 + consumer接口
>
> - 访问http://localhost:9000/abc8000/consumer/get/1

## 屏蔽微服务名称

原来的[微服务名称]()方式还是可以访问，所以必须屏蔽。

- 修改配置文件

  - 在配置文件中添加如下配置。

  - ```yaml
    zuul:
    	routes:
    		#指定路由规则
    		abcmsc-consumer-depart-8080: /abc8000/**
    		abcmsc-consumer-depart-8090: /abc9000/**
    	#屏蔽所有微服务名称
    	ignore-services: "*"
    		
    ```

> 如何测试：
>
> - 访问路径 = zuul地址 + [consumer微服务名称的别称]() + consumer接口
>
> - 访问http://localhost:9000/abcmsc-consumer-depart-8080/consumer/depart/get/1失效

## 路由前辍 

在配置路由策略时，可以为路由路径配置一个统一的前辍，以便为请求归类。

- 修改配置文件

  - 在配置文件中添加如下配置。

  - ```yaml
    zuul:
    	##指定前缀
    	prefix: /abc
    	routes:
    		#指定路由规则
    		abcmsc-consumer-depart-8080: /abc8000/**
    		abcmsc-consumer-depart-8090: /abc9000/**
    	#屏蔽所有微服务名称
    	ignore-services: "*"
    ```

> 如何测试：
>
> - 访问路径 = zuul地址 + 前缀 + [consumer微服务名称的别称]() + consumer接口
>
> - 访问http://localhost:9000/abc/abc8000/consumer/depart/get/1

## 路径屏蔽

 可以指定屏蔽掉的路径 URI，即只要用户请求中包含指定的 URI 路径，那么该请求将无法访问到指定的服务。通过该方式可以限制用户的权限。

- 修改配置文件

  - 在配置文件中添加如下配置。

  - ```yaml
    zuul:
    	##指定前缀
    	prefix: /abc
    	routes:
    		#指定路由规则
    		abcmsc-consumer-depart-8080: /abc8000/**
    		abcmsc-consumer-depart-8090: /abc9000/**
    	#屏蔽所有微服务名称
    	ignore-services: "*"
    	#屏蔽指定URI
    	ignore-patterns: /**/list/**
    ```

> 如何测试：
>
> - 访问http://localhost:9000/abc/abc8000/consumer/depart/list失效



## 敏感请求头屏蔽

默认情况下，像 Cookie、Set-Cookie 等敏感请求头信息会被 zuul 屏蔽掉，我们可以将这些默认屏蔽去掉，当然，也可以添加要屏蔽的请求头。

- 修改配置文件

  - 在配置文件中添加如下配置。

  - ```yaml
    zuul:
    	##指定前缀
    	prefix: /abc
    	routes:
    		#指定路由规则
    		abcmsc-consumer-depart-8080: /abc8000/**
    		abcmsc-consumer-depart-8090: /abc9000/**
    	#屏蔽所有微服务名称
    	ignore-services: "*"
    	#屏蔽指定URI
    	ignore-patterns: /**/list/**
    	#设置需要屏蔽的敏感头
    	sensitive-headers: token, set-cookies,cookies
    ```



# 对Conumser负载均衡

查看zuul的依赖，有Hystrix和ribbon，所以可以负载均衡。

用户提交的请求被路由到一个指定的微服务中，若该微服务名称的主机有多个，则默认采用负载均衡策略是轮询。

- 创建多个conusmer

  - 微服务名称都设置为abc-consumer-depart

- Zuul server

  - 修改配置文件，指定路由规则，则自动进行负载均衡

    - ```yaml
      zuul:
        ##指定前缀
        prefix: /abc
        routes:
          #指定路由规则
          abcmsc-consumer-depart: /abc123/**
      ```

    - 测试访问http://localhost:9000/abc/abc123/consumer/depart/list发现负载均衡

  - 修改负载均衡策略，使用随机策略

    - 启动类

    - ```java
      @EnableZuulProxy
      @SpringBootApplication{
      	....
      	@Bean
      	public IRule getRule(){
      		return new RandomRule();
      	}
      }
      ```

      

# 对Conumser服务降级

当消费者调用提供者时由于各种原因出现无法调用的情况时，消费者可以进行服务降级。那么，若客户端通过网关调用消费者无法调用时, 同样可以降级。

- 在Zuul server中定义 **fallback** 类

  - 可以对指定微服务降级，通过getRoute()方法 + 在fallbackResponse()方法中对route做判断

  - ```java
    @Component
    public class ConsumerFallback implements FallbackProvider {
        @Override
        public String getRoute() {
            // 对指定的微服务进行降级
            // return "abcmsc-consumer-depart-8080";
            // 指定对所有微服务进行降级
            return "*";
        }
    
        @Override
        public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
            // 若微服务不是abcmsc-consumer-depart-8080，则不进行降级
            // if (!"abcmsc-consumer-depart-8080".equals(route)) {
            //     return null;
            // }
    
            // 仅对abcmsc-consumer-depart-8080进行降级
            return new ClientHttpResponse() {
                @Override
                public HttpHeaders getHeaders() {
                    HttpHeaders headers = new HttpHeaders();
                    headers.setContentType(MediaType.APPLICATION_JSON);
                    return headers;
                }
    
                @Override
                public InputStream getBody() throws IOException {
                    String msg = "fallback:" + route;
                    return new ByteArrayInputStream(msg.getBytes());
                }
    
                @Override
                public HttpStatus getStatusCode() throws IOException {
                    return HttpStatus.SERVICE_UNAVAILABLE;
                }
    
                @Override
                public int getRawStatusCode() throws IOException {
                    return HttpStatus.SERVICE_UNAVAILABLE.value();
                }
    
                @Override
                public String getStatusText() throws IOException {
                    return HttpStatus.SERVICE_UNAVAILABLE.getReasonPhrase();
                }
    
                @Override
                public void close() {
                    // 写资源释放代码
                }
            };
        }
    }
    ```

    

# 请求过滤 

在服务路由之前、中、后，可以对请求进行过滤，使其只能访问它应该访问到的资源，增强安全性。此时需要通过 ZuulFilter 过滤器来实现对外服务的安全控制。

<img src="http://yuwb.pub/usr/uploads/2018/11/1832100691.png" style="zoom:50%;" />

- 需求

  - 该过滤的条件是，只有请求参数携带有 user 的请求才可访问/abc8080 工程，否则返回 401，未授权。当然，对/abc8090 工程的访问没有限制。简单来说就是，只有当访问/abc8080 且 user 为空时是通不过过滤的，其它请求都可以。

- 定义 **RouteFilter** 类

  - ```java
    @Component
    public class RouteFilter extends ZuulFilter {
        @Override
        public String filterType() {
            // "pre",进行路由之前过滤
            return FilterConstants.PRE_TYPE;
        }
    
        @Override
        public int filterOrder() {
          	// 越小，越靠前执行
            return -5;
        }
    
      	//为true表示通过
        @Override
        public boolean shouldFilter() {
            // 获取当前的请求上下文对象
            RequestContext context = RequestContext.getCurrentContext();
            // 从请求上下文中获取当前请求信息
            HttpServletRequest request = context.getRequest();
            String user = request.getParameter("user");
            String uri = request.getRequestURI();
            if (uri.contains("/abc8080") && StringUtils.isEmpty(user)) {
                // 指定当前请求未通过zuul过滤，默认值为true
                context.setSendZuulResponse(false);
                context.setResponseStatusCode(HttpStatus.UNAUTHORIZED.value());
                return false;
            }
            return true;
        }
    
        @Override
        public Object run() throws ZuulException {
            System.out.println("通过过滤");
            return null;
        }
    }
    ```

    

# 令牌桶限流

通过对请求限流的方式避免系统遭受“雪崩之灾”。

我们下面的代码使用 Guava 库的 RateLimit 完成限流的，而其底层使用的是令牌桶算法 实现的限流，所以我们先来学习一下令牌桶限流算法。

## 原理

- 令牌桶算法
- 漏斗限流算法

实现

- 修改**RouteFilter** 类
  - 重写逻辑代码，即shouldFilter()方法
  - 具体代码（略）

# 多维请求限流 

## 原理

使用 Guava 的 RateLimit 令牌桶算法可以实现对请求的限流，但其限流粒度有些大。有 个老外使用路由过滤，针对 Zuul 编写了一个限流库(spring-cloud-zuul-ratelimit)，提供多种细 粒度限流策略，在导入该依赖后我们就可以直接使用了。

其限流策略，即限流查验的对象类型有:

- user:针对用户的限流，即对单位时间窗内经过网关的用户数量的限制。
- origin:针对客户端IP的限流，即对单位时间窗内经过网关的IP数量的限制。 
- url:针对请求URL的限流，即对单位时间窗内经过网关的URL数量的限制。

[对于某个请求，只要不符合任意一项，都无法通过。]()

## 实现

- 删除RouteFilter类

- 添加依赖

  - ```xml
    <!-- spring-cloud-zuul-ratelimit 依赖 --> 
    <dependency>
      <groupId>com.marcosbarbero.cloud</groupId> 
      <artifactId>spring-cloud-zuul-ratelimit</artifactId》
      <version>2.0.5.RELEASE</version>
    </dependency>
    ```

- 修改配置文件

  - ```yaml
    zuul:
      routes:
        abcmsc-consumer-depart-8080: /abc8080/**
        abcmsc-consumer-depart-8090: /abc8090/**
    
      ratelimit:
        enabled: true  # 开启限流
        # 设置限流策略
        # 在一个单位时间窗内通过该zuul的用户数量、ip数量及url数量，都不能超过3个
        default-policy:
          quota: 1   # 指定限流的时间窗数量
          refresh-interval: 3    # 指定单位时间窗大小，单位秒
          limit: 3  # 在指定的单位时间窗内启用限流的限定值
          type: user,origin,url   # 指定限流查验的类型
    ```

- 添加异常处理页面

  - 在 src/main/resources 目录下再定义新的目录 public/error，必须是这个目录名称。然后 在该目录中定义一个异常处理页面，名称必须是异常状态码，扩展名必须为 html。

# 灰度发布

## 原理

(**1**) 什么是灰度发布 灰度发布，又名金丝雀发布，是系统迭代更新、平滑过渡的一种上线发布方式。

(**2**) **Zuul** 灰度发布原理 生产环境中，可以实现灰度发布的技术很多，我们这里要讲的是 zuul 对于灰度发布的实现。而其实现也是基于 Eureka 元数据的(自定义元数据)。

## consumer

- 修改配置文件

  - 添加Eureka元数据

  - ```yaml
    eureka:
    	client:
    		....略
    		
    	instance:
      	metadata-map: #指定添加的元数据
      		host-mark: running-host #生产的机器
      		#host-mark: gray-host #灰度发布的机器
    ```

    

## Zuul server

### 原理1

[ browser携带gray-host请求头, Zuul通过过滤器检测请求头；若携带gray-host请求头，那么把该请求路由到灰度发布的机器。]()

- 添加依赖: 不是Spring cloud官网开发的

  - ```xml
    <dependency>
      <groupId>io.jmnarloch</groupId> 
      <artifactId>ribbon-discovery-filter-spring-cloud-starter</artifactId>
      <version>2.1.0</version>
    </dependency>
    ```

    

- 定义过滤器

  - 通过过滤器实现灰度发布

  - ```java
    mport io.jmnarloch.spring.cloud.ribbon.support.RibbonFilterContextHolder;
    import org.springframework.cloud.netflix.zuul.filters.support.FilterConstants;
    import org.springframework.util.StringUtils;
    
    import javax.servlet.http.HttpServletRequest;
    
    // @Component
    public class GrayFilter extends ZuulFilter {
        @Override
        public String filterType() {
            return FilterConstants.PRE_TYPE;
        }
    
        @Override
        public int filterOrder() {
            return -5;
        }
    
        @Override
        public boolean shouldFilter() {
            // 所有请求都通过zuul过滤
            return true;
        }
    
        @Override
        public Object run() throws ZuulException {
            RequestContext context = RequestContext.getCurrentContext();
            HttpServletRequest request = context.getRequest();
            // 获取指定的请求头信息，该头信息在浏览器提交请求时携带，用于区分该请求要被路由到哪个主机处理
            String mark = request.getHeader("gray-mark");
            // 默认将请求路由到running-host上
            RibbonFilterContextHolder.getCurrentContext().add("host-mark", "running-host");
            // 若mark的值不为空且值为enable，则将请求路由到gray-host，其它请求会路由到默认的running-host
            if (!StringUtils.isEmpty(mark) && "enable".equals(mark)) {
                RibbonFilterContextHolder.getCurrentContext().add("host-mark", "gray-host");
            }
            return null;
        }
    }
    ```

    

### 原理2

原理1得需要2套页面，一个不带gray-host，一个带gray-host，所以不方便。现在改为交替访问gray-host。

- 定义过滤器

  - 通过全局变量Boolean flag，每次交互访问gray-host

  - ```java
    @Component
    public class GrayFilter2 extends ZuulFilter {
        // 定义一个原子布尔变量，是为了解决当前单例中全局变量的线程安全问题
        private AtomicBoolean flag = new AtomicBoolean(true);
    
        @Override
        public String filterType() {
            return FilterConstants.PRE_TYPE;
        }
    
        @Override
        public int filterOrder() {
            return -5;
        }
    
        @Override
        public boolean shouldFilter() {
            // 所有请求都通过zuul过滤
            return true;
        }
    
        @Override
        public Object run() throws ZuulException {
            RequestContext context = RequestContext.getCurrentContext();
            HttpServletRequest request = context.getRequest();
    
            // 根据布尔变量的值的不同，路由到不同的主机，然后再将布尔值取反
            if (flag.get()) {
                RibbonFilterContextHolder.getCurrentContext().add("host-mark", "running-host");
                flag.set(false);
            } else {
                RibbonFilterContextHolder.getCurrentContext().add("host-mark", "gray-host");
                flag.set(true);
            }
    
            return null;
        }
    }
    ```

    

# **Zuul**的高可用

Zuul 的高可用非常关键，因为外部请求到后端微服务的流量都会经过 Zuul。故而在生产 环境中，我们一般都需要部署高可用的 Zuul 以避免单点故障。

作为整个系统入口路由的高可用，需要借助额外的负载均衡器来实现，例如 Nginx、 HAProxy、F5 等。在 Zuul 集群的前端部分部署负载均衡服务器。Zuul 客户端将请求发送到负 载均衡器，负载均衡器将请求转发到其代理的其中一个 Zuul 节点。这样，就可以实现 Zuul 的高可用。