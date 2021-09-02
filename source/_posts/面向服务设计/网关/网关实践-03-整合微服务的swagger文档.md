---
title: 网关实践-01-整合微服务的swagger文档
date: 2021-05-16 12:32:03
tags:
---

由于微服务的划分，使用Swagger生成的接口文档也随之拆散，前端同事不得不把每个微服务的接口文档保存为浏览器标签，方便快速切换。在引入网关之后我们想改善这个问题，统一多个微服务接口文档的入口。

### 添加依赖

基于Spring Cloud Gateway开发微服务网关，使用的是Project Reactor、WebFlux提供的API。而在网关项目中整合Swagger实际就是在WebFlux项目中整合Swagger。

由于我们项目使用的Spring Boot版本号是2.3.0.RELEASE，因此我们选择的Swagger版本号为2.10.x。

```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-spring-webflux</artifactId>
    <version>2.10.5</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.10.5</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.10.5</version>
</dependency>
```

### 为网关添加Swagger

接着需要为WebFlux添加静态资源文件访问路径映射，即添加ResourceWebHandler。

```java
@EnableSwagger2WebFlux
@Configuration
public class WebfluxConfiguration implements WebFluxConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/swagger-ui.html**")
                .addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");
    }

}
```

为一个WebFlux项目配置了Swagger功能，现在打开浏览器访问`/swagger-ui.html#/`看到的只是Swagger为网关项目生成的接口文档。

### 将多个微服务Swagger路径注册到网关

#### 方案一

##### 网关中自定义SwaggerResourcesProvider

在网关项目中自定义SwaggerResourcesProvider替换Swagger提供的。

自定义SwaggerResourcesProvider实现SwaggerResourcesProvider接口的get方法，方法可返回多个SwaggerResource，每个SwaggerResource对应每个微服务，我们可以过滤掉网关自身的，代码如下。

```java
@Primary
@Profile({"dev", "test"}) // 仅本地测试、测试环境开启
@Component
public class GatewaySwaggerResourcesProvider implements SwaggerResourcesProvider{
    @Override
    public List<SwaggerResource> get() {
        List<SwaggerResource> swaggerResources = new ArrayList<>();
        routeProperties.getRoutes().forEach(routeDefinition -> {
            String routeId = routeDefinition.getId();
            String baseUrl;
            if (isLocalDebug) {
                 baseUrl = "http://127.0.0.1:" + routeDefinition.getUri().getPort();
            } else {
                 baseUrl = ${网关域名} + "/" + ${路由前缀};
            }
            swaggerResources.add(getSwaggerResources(routeId, baseUrl));
        });
        return swaggerResources;
    }

    private SwaggerResource getSwaggerResources(String name, String baseUrl) {
            SwaggerResource resource = new SwaggerResource();
            resource.setName(name);
            resource.setLocation(baseUrl + "/v2/api-docs");
            resource.setSwaggerVersion("2.0");
            return resource;
    }
}
```

这些SwaggerResource就是我们在Swagger ui看到的"select a definition"的一个个选项

关于SwaggerResource：

- name：swagger资源名称，微服务名称，也是我们在Swagger ui看到的选项的名称；
- location：对应微服务接口文档的`/v2/api-docs` API的url（即`https://{网关host}/{路由到该应用的路由规则}/v2/api-docs`）

`GatewaySwaggerResourcesProvider`中的下面这段代码只是拼接接口文档的baseUrl，满足“baseUrl+/v2/api-docs”能够直接在浏览器访问。

由于SwaggerResource配置的location是由[前端]()直接发起请求的，而不是由网关发起请求获取再响应给前端，因此[需要在网关为每个微服务配置“`/v2/api-docs`”接口的路由规则]()。

##### 微服务中配置支持跨域请求

最后还需要**为其它微服务配置支持跨域请求**，否则Swagger前端无法调用SwaggerResource配置的location向后端服务发起`“/v2/api-docs”`接口请求。在其它微服务中添加跨域请求配置如下（**注意：不是在网关添加！**）。

```java
@Profile({"dev", "test"}) // 仅测试环境
@ConditionalOnClass(WebMvcConfigurer.class)
@Configuration
public class CorsAutoConfiguration {

    private CorsConfiguration buildConfig() {
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.addAllowedOrigin("*");
        corsConfiguration.addAllowedHeader("*");
        corsConfiguration.addAllowedMethod("*");
        return corsConfiguration;
    }

    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", buildConfig());
        return new CorsFilter(source);
    }

}
```

#### 方案二

虽然方案一可行，但弊端是要为每个微服务添加Swagger的“`/v2/api-docs`”接口路由配置。

方案二可以通过在网关**代理后端每个微服务的“`/v2/api-docs`”接口请求**实现。

简单说，

- 网关的自定义SwaggerResource，存储映射：[网关代理的`/v2/api-docs`接口 routeId]()与[微服务的swagger地址]()，如指向用户中心的SwaggerResource配置的location为“`https://网关host/proxySwagger/v2/api-docs?routeId=usercenter`”，
- 网关提供`/proxySwagger/v2/api-docs`接口，实现根据参数`routeId`向后端微服务发起“`/v2/api-docs`”接口请求，并将响应结果直接响应给前端。

##### 网关中自定义SwaggerResourcesProvider

`GatewaySwaggerResourcesProvider`代码实现如下：

```java
@Primary
@Profile({"dev", "test"}) // 仅本地测试、测试环境开启
@Component
public class GatewaySwaggerResourcesProvider implements SwaggerResourcesProvider {

    @Resource
    private RouteProperties routeProperties;
    @Value("${server.domain:http://127.0.0.1:8600}")
    private String gatewayDomain;

    @Override
    public List<SwaggerResource> get() {
        List<SwaggerResource> swaggerResources = new ArrayList<>();
        // 遍历路由定义
        routeProperties.getRoutes().forEach(routeDefinition -> {
            String routeId = routeDefinition.getId();
            // 参数1:路由的id
            // 参数2:网关的域名
            swaggerResources.add(getSwaggerResources(routeId, gatewayDomain));
        });
        return swaggerResources;
    }

    private SwaggerResource getSwaggerResources(String routeId, String baseUrl) {
        SwaggerResource resource = new SwaggerResource();
        resource.setName(routeId);
        resource.setLocation(baseUrl + "/proxySwagger/v2/api-docs?routeId=" + routeId);
        resource.setSwaggerVersion("2.0");
        return resource;
    }

}
```

##### 网关开发swagger proxy接口

由网关提供`/v2/api-docs`的代理接口`/proxySwagger/v2/api-docs?routeId=${routeId}`，代码如下：

```java
@Profile({"dev", "test"})
@RestController
@RequestMapping("/proxySwagger")
public class GatewayApiDocsController {

    @Resource
    private RouteProperties routeProperties;
    private WebClient webClient;
    private boolean isLocalDebug;

    @PostConstruct
    public void init() {
        webClient = WebClient.create();
        isLocalDebug = ....;// dev: true, test: false
    }

    @GetMapping("/v2/api-docs")
    public Mono<String> proxyApiDocs(@RequestParam("routeId") String routeId) {
        // 根据路由id获取路由配置
        RouteDefinition routeDefinition = routeProperties.getRoutes().stream()
                .filter(rd -> rd.getId().equals(routeId))
                .findFirst().get();
        String baseUrl;
        URI routeUri = routeDefinition.getUri();
        if (isLocalDebug) {
            // 本地debug
            baseUrl = "http://127.0.0.1";
        } else {
            baseUrl = "http://" + routeUri.getHost();
        }
        if (routeUri.getPort() > 0) {
            baseUrl += (":" + routeUri.getPort());
        }
        // 转发给后端微服务
        return webClient.get()
                .uri(baseUrl + "/v2/api-docs")
                .retrieve()
                .bodyToMono(String.class);
    }
    
}
```

- 注意：由于我们项目是部署在k8s上的，我们去掉了服务注册/发现，所有路由规则配置的uri就已经是k8s集群内部pod可以直接访问的。



### 总结

这种方案相比第一种方案优点在于：

- 严格统一流量入口，不让Swagger前端绕过网关直接访问后端微服务的接口；
- 不必要求每个微服务都配置支持跨域请求；
- 不必为Swagger的api配置路由规则。