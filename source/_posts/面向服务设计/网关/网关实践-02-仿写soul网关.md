---
title: 网关实践-02-仿写soul网关
date: 2021-05-10 00:48:10
tags:
---



我的网关 ship-gate 核心功能基本都已完成，最大的缺陷就是前端功底太差没有管理后台 😤。

## 设计

### 吞吐量

网关是所有请求的入口，所以要求有很高的吞吐量，为了实现这点可以使用请求异步化来解决。<!--提到异步化，我就考虑链接超时问题--> 目前一般有以下两种方案：

- Tomcat/Jetty+NIO+Servlet3
  - Servlet3 已经支持异步

- Netty+NIO
  - Netty 为高并发而生，目前唯品会的网关使用这个策略，在唯品会的技术文章中在相同的情况下 Netty 是每秒 30w+的吞吐量，Tomcat 是 13w+,但是 Netty 需要自己处理 HTTP 协议
  - Soul 网关是基于 Spring WebFlux（底层 Netty），不用太关心 HTTP 协议的处理，于是决定也用 Spring WebFlux。

### 拓展性

比如 Netflix Zuul 有 preFilters，postFilters 等在不同的阶段方便处理不同的业务，基于责任链模式将请求进行链式处理即可实现。

在微服务架构下，服务都会进行多实例部署来保证高可用，请求到达网关时，网关需要根据 URL 找到所有可用的实例，这时就需要服务注册和发现功能，即注册中心。

现在流行的注册中心有 Apache 的 Zookeeper 和阿里的 Nacos 两种（consul 有点小众），因为之前写 RPC 框架时已经用过了 Zookeeper，所以这次就选择了 Nacos。

### 需求清单

开发一个具备哪些特性的网关

- 自定义路由规则

  可基于 version 的路由规则设置，路由对象包括 DEFAUL,HEADER 和 QUERY 三种，匹配方式包括=、regex、like 三种。

- 跨语言

  HTTP 协议天生跨语言

- 高性能

  Netty 本身就是一款高性能的通信框架，同时 server 将一些路由规则等数据缓存到 JVM 内存避免请求 admin 服务。

- 高可用

  支持集群模式防止单节点故障，无状态。

- 灰度发布

  灰度发布是指在黑与白之间，能够平滑过渡的一种发布方式。在其上可以进行 A/B testing，即让一部分用户继续用产品特性 A，一部分用户开始用产品特性 B，如果用户对 B 没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到 B 上面来。通过特性一可以实现。

- 接口鉴权

  基于责任链模式，用户开发自己的鉴权插件即可。

- 负载均衡

  支持多种负载均衡算法，如随机，轮询，加权轮询等。利用 SPI 机制可以根据配置进行动态加载。

### 架构设计

在参考了一些优秀的网关 Zuul,Spring Cloud Gateway,Soul 后，将项目划分为以下几个模块。

| 名称                            | 描述                                   |
| :------------------------------ | :------------------------------------- |
| ship-admin                      | 后台管理界面，配置路由规则等           |
| ship-server                     | 网关服务端，核心功能模块               |
| ship-client-spring-boot-starter | 网关客户端，自动注册服务信息到注册中心 |
| ship-common                     | 一些公共的代码，如 pojo，常量等。      |

它们之间的关系如图：

![网关设计](https://img2020.cnblogs.com/blog/1167086/202101/1167086-20210102204020550-70612604.png)

**注意**：这张图与实际实现有点出入，Nacos push 到本地缓存的那个环节没有实现，目前只有 ship-sever 定时轮询 pull 的过程。ship-admin 从 Nacos 获取注册服务信息的过程，也改成了 ServiceA 启动时主动发生 HTTP 请求通知 ship-admin。

## 实现

### ship-client-spring-boot-starter

首先创建一个 spring-boot-starter 命名为 ship-client-spring-boot-starter

其核心类 **AutoRegisterListener** 就是在项目启动时做了两件事：

1.将服务信息注册到 Nacos 注册中心

2.通知 ship-admin 服务上线了并注册下线 hook。

代码如下：

```java
/**
 * Created by 2YSP on 2020/12/21
 */
public class AutoRegisterListener implements ApplicationListener<ContextRefreshedEvent> {

    private final static Logger LOGGER = LoggerFactory.getLogger(AutoRegisterListener.class);

    private volatile AtomicBoolean registered = new AtomicBoolean(false);

    private final ClientConfigProperties properties;

    @NacosInjected
    private NamingService namingService;

    @Autowired
    private RequestMappingHandlerMapping handlerMapping;

    private final ExecutorService pool;

    /**
     * url list to ignore
     */
    private static List<String> ignoreUrlList = new LinkedList<>();

    static {
        ignoreUrlList.add("/error");
    }

    public AutoRegisterListener(ClientConfigProperties properties) {
        if (!check(properties)) {
            LOGGER.error("client config port,contextPath,appName adminUrl and version can't be empty!");
            throw new ShipException("client config port,contextPath,appName adminUrl and version can't be empty!");
        }
        this.properties = properties;
        pool = new ThreadPoolExecutor(1, 4, 0, TimeUnit.SECONDS, new LinkedBlockingQueue<>());
    }

    /**
     * check the ClientConfigProperties
     *
     * @param properties
     * @return
     */
    private boolean check(ClientConfigProperties properties) {
        if (properties.getPort() == null || properties.getContextPath() == null
                || properties.getVersion() == null || properties.getAppName() == null
                || properties.getAdminUrl() == null) {
            return false;
        }
        return true;
    }


    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (!registered.compareAndSet(false, true)) {
            return;
        }
        doRegister();
        registerShutDownHook();
    }

    /**
     * send unregister request to admin when jvm shutdown
     */
    private void registerShutDownHook() {
        final String url = "http://" + properties.getAdminUrl() + AdminConstants.UNREGISTER_PATH;
        final UnregisterAppDTO unregisterAppDTO = new UnregisterAppDTO();
        unregisterAppDTO.setAppName(properties.getAppName());
        unregisterAppDTO.setVersion(properties.getVersion());
        unregisterAppDTO.setIp(IpUtil.getLocalIpAddress());
        unregisterAppDTO.setPort(properties.getPort());
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            OkhttpTool.doPost(url, unregisterAppDTO);
            LOGGER.info("[{}:{}] unregister from ship-admin success!", unregisterAppDTO.getAppName(), unregisterAppDTO.getVersion());
        }));
    }

    /**
     * register all interface info to register center
     */
    private void doRegister() {
        Instance instance = new Instance();
        instance.setIp(IpUtil.getLocalIpAddress());
        instance.setPort(properties.getPort());
        instance.setEphemeral(true);
        Map<String, String> metadataMap = new HashMap<>();
        metadataMap.put("version", properties.getVersion());
        metadataMap.put("appName", properties.getAppName());
        instance.setMetadata(metadataMap);
        try {
            namingService.registerInstance(properties.getAppName(), NacosConstants.APP_GROUP_NAME, instance);
        } catch (NacosException e) {
            LOGGER.error("register to nacos fail", e);
            throw new ShipException(e.getErrCode(), e.getErrMsg());
        }
        LOGGER.info("register interface info to nacos success!");
        // send register request to ship-admin
        String url = "http://" + properties.getAdminUrl() + AdminConstants.REGISTER_PATH;
        RegisterAppDTO registerAppDTO = buildRegisterAppDTO(instance);
        OkhttpTool.doPost(url, registerAppDTO);
        LOGGER.info("register to ship-admin success!");
    }


    private RegisterAppDTO buildRegisterAppDTO(Instance instance) {
        RegisterAppDTO registerAppDTO = new RegisterAppDTO();
        registerAppDTO.setAppName(properties.getAppName());
        registerAppDTO.setContextPath(properties.getContextPath());
        registerAppDTO.setIp(instance.getIp());
        registerAppDTO.setPort(instance.getPort());
        registerAppDTO.setVersion(properties.getVersion());
        return registerAppDTO;
    }
}
```

### ship-server

ship-sever 项目主要包括了两个部分内容， 1.请求动态路由的主流程 2.本地缓存数据和 ship-admin 及 nacos 同步，这部分在后面 3.3 再讲。

ship-server 实现动态路由的原理是利用 WebFilter 拦截请求，然后将请求教给 plugin chain 去链式处理。

PluginFilter 根据 URL 解析出 appName，然后将启用的 plugin 组装成 plugin chain。

```
public class PluginFilter implements WebFilter {

    private ServerConfigProperties properties;

    public PluginFilter(ServerConfigProperties properties) {
        this.properties = properties;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String appName = parseAppName(exchange);
        if (CollectionUtils.isEmpty(ServiceCache.getAllInstances(appName))) {
            throw new ShipException(ShipExceptionEnum.SERVICE_NOT_FIND);
        }
        PluginChain pluginChain = new PluginChain(properties, appName);
        pluginChain.addPlugin(new DynamicRoutePlugin(properties));
        pluginChain.addPlugin(new AuthPlugin(properties));
        return pluginChain.execute(exchange, pluginChain);
    }

    private String parseAppName(ServerWebExchange exchange) {
        RequestPath path = exchange.getRequest().getPath();
        String appName = path.value().split("/")[1];
        return appName;
    }
}
```

PluginChain 继承了 AbstractShipPlugin 并持有所有要执行的插件。

```
/**
 * @Author: Ship
 * @Description:
 * @Date: Created in 2020/12/25
 */
public class PluginChain extends AbstractShipPlugin {
    /**
     * the pos point to current plugin
     */
    private int pos;
    /**
     * the plugins of chain
     */
    private List<ShipPlugin> plugins;

    private final String appName;

    public PluginChain(ServerConfigProperties properties, String appName) {
        super(properties);
        this.appName = appName;
    }

    /**
     * add enabled plugin to chain
     *
     * @param shipPlugin
     */
    public void addPlugin(ShipPlugin shipPlugin) {
        if (plugins == null) {
            plugins = new ArrayList<>();
        }
        if (!PluginCache.isEnabled(appName, shipPlugin.name())) {
            return;
        }
        plugins.add(shipPlugin);
        // order by the plugin's order
        plugins.sort(Comparator.comparing(ShipPlugin::order));
    }

    @Override
    public Integer order() {
        return null;
    }

    @Override
    public String name() {
        return null;
    }

    @Override
    public Mono<Void> execute(ServerWebExchange exchange, PluginChain pluginChain) {
        if (pos == plugins.size()) {
            return exchange.getResponse().setComplete();
        }
        return pluginChain.plugins.get(pos++).execute(exchange, pluginChain);
    }

    public String getAppName() {
        return appName;
    }

}
```

AbstractShipPlugin 实现了 ShipPlugin 接口，并持有 ServerConfigProperties 配置对象。

```
public abstract class AbstractShipPlugin implements ShipPlugin {

    protected ServerConfigProperties properties;

    public AbstractShipPlugin(ServerConfigProperties properties) {
        this.properties = properties;
    }
}
```

ShipPlugin 接口定义了所有插件必须实现的三个方法 order(),name()和 execute()。

```
public interface ShipPlugin {
    /**
     * lower values have higher priority
     *
     * @return
     */
    Integer order();

    /**
     * return current plugin name
     *
     * @return
     */
    String name();

    Mono<Void> execute(ServerWebExchange exchange,PluginChain pluginChain);

}
```

DynamicRoutePlugin 继承了抽象类 AbstractShipPlugin，包含了动态路由的主要业务逻辑。

```
/**
 * @Author: Ship
 * @Description:
 * @Date: Created in 2020/12/25
 */
public class DynamicRoutePlugin extends AbstractShipPlugin {

    private final static Logger LOGGER = LoggerFactory.getLogger(DynamicRoutePlugin.class);

    private static WebClient webClient;

    private static final Gson gson = new GsonBuilder().create();

    static {
        HttpClient httpClient = HttpClient.create()
                .tcpConfiguration(client ->
                        client.doOnConnected(conn ->
                                conn.addHandlerLast(new ReadTimeoutHandler(3))
                                        .addHandlerLast(new WriteTimeoutHandler(3)))
                                .option(ChannelOption.TCP_NODELAY, true)
                );
        webClient = WebClient.builder().clientConnector(new ReactorClientHttpConnector(httpClient))
                .build();
    }

    public DynamicRoutePlugin(ServerConfigProperties properties) {
        super(properties);
    }

    @Override
    public Integer order() {
        return ShipPluginEnum.DYNAMIC_ROUTE.getOrder();
    }

    @Override
    public String name() {
        return ShipPluginEnum.DYNAMIC_ROUTE.getName();
    }

    @Override
    public Mono<Void> execute(ServerWebExchange exchange, PluginChain pluginChain) {
        String appName = pluginChain.getAppName();
        ServiceInstance serviceInstance = chooseInstance(appName, exchange.getRequest());
//        LOGGER.info("selected instance is [{}]", gson.toJson(serviceInstance));
        // request service
        String url = buildUrl(exchange, serviceInstance);
        return forward(exchange, url);
    }

    /**
     * forward request to backend service
     *
     * @param exchange
     * @param url
     * @return
     */
    private Mono<Void> forward(ServerWebExchange exchange, String url) {
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();
        HttpMethod method = request.getMethod();

        WebClient.RequestBodySpec requestBodySpec = webClient.method(method).uri(url).headers((headers) -> {
            headers.addAll(request.getHeaders());
        });

        WebClient.RequestHeadersSpec<?> reqHeadersSpec;
        if (requireHttpBody(method)) {
            reqHeadersSpec = requestBodySpec.body(BodyInserters.fromDataBuffers(request.getBody()));
        } else {
            reqHeadersSpec = requestBodySpec;
        }
        // nio->callback->nio
        return reqHeadersSpec.exchange().timeout(Duration.ofMillis(properties.getTimeOutMillis()))
                .onErrorResume(ex -> {
                    return Mono.defer(() -> {
                        String errorResultJson = "";
                        if (ex instanceof TimeoutException) {
                            errorResultJson = "{\"code\":5001,\"message\":\"network timeout\"}";
                        } else {
                            errorResultJson = "{\"code\":5000,\"message\":\"system error\"}";
                        }
                        return ShipResponseUtil.doResponse(exchange, errorResultJson);
                    }).then(Mono.empty());
                }).flatMap(backendResponse -> {
                    response.setStatusCode(backendResponse.statusCode());
                    response.getHeaders().putAll(backendResponse.headers().asHttpHeaders());
                    return response.writeWith(backendResponse.bodyToFlux(DataBuffer.class));
                });
    }

    /**
     * weather the http method need http body
     *
     * @param method
     * @return
     */
    private boolean requireHttpBody(HttpMethod method) {
        if (method.equals(HttpMethod.POST) || method.equals(HttpMethod.PUT) || method.equals(HttpMethod.PATCH)) {
            return true;
        }
        return false;
    }

    private String buildUrl(ServerWebExchange exchange, ServiceInstance serviceInstance) {
        ServerHttpRequest request = exchange.getRequest();
        String query = request.getURI().getQuery();
        String path = request.getPath().value().replaceFirst("/" + serviceInstance.getAppName(), "");
        String url = "http://" + serviceInstance.getIp() + ":" + serviceInstance.getPort() + path;
        if (!StringUtils.isEmpty(query)) {
            url = url + "?" + query;
        }
        return url;
    }


    /**
     * choose an ServiceInstance according to route rule config and load balancing algorithm
     *
     * @param appName
     * @param request
     * @return
     */
    private ServiceInstance chooseInstance(String appName, ServerHttpRequest request) {
        List<ServiceInstance> serviceInstances = ServiceCache.getAllInstances(appName);
        if (CollectionUtils.isEmpty(serviceInstances)) {
            LOGGER.error("service instance of {} not find", appName);
            throw new ShipException(ShipExceptionEnum.SERVICE_NOT_FIND);
        }
        String version = matchAppVersion(appName, request);
        if (StringUtils.isEmpty(version)) {
            throw new ShipException("match app version error");
        }
        // filter serviceInstances by version
        List<ServiceInstance> instances = serviceInstances.stream().filter(i -> i.getVersion().equals(version)).collect(Collectors.toList());
        //Select an instance based on the load balancing algorithm
        LoadBalance loadBalance = LoadBalanceFactory.getInstance(properties.getLoadBalance(), appName, version);
        ServiceInstance serviceInstance = loadBalance.chooseOne(instances);
        return serviceInstance;
    }


    private String matchAppVersion(String appName, ServerHttpRequest request) {
        List<AppRuleDTO> rules = RouteRuleCache.getRules(appName);
        rules.sort(Comparator.comparing(AppRuleDTO::getPriority).reversed());
        for (AppRuleDTO rule : rules) {
            if (match(rule, request)) {
                return rule.getVersion();
            }
        }
        return null;
    }


    private boolean match(AppRuleDTO rule, ServerHttpRequest request) {
        String matchObject = rule.getMatchObject();
        String matchKey = rule.getMatchKey();
        String matchRule = rule.getMatchRule();
        Byte matchMethod = rule.getMatchMethod();
        if (MatchObjectEnum.DEFAULT.getCode().equals(matchObject)) {
            return true;
        } else if (MatchObjectEnum.QUERY.getCode().equals(matchObject)) {
            String param = request.getQueryParams().getFirst(matchKey);
            if (!StringUtils.isEmpty(param)) {
                return StringTools.match(param, matchMethod, matchRule);
            }
        } else if (MatchObjectEnum.HEADER.getCode().equals(matchObject)) {
            HttpHeaders headers = request.getHeaders();
            String headerValue = headers.getFirst(matchKey);
            if (!StringUtils.isEmpty(headerValue)) {
                return StringTools.match(headerValue, matchMethod, matchRule);
            }
        }
        return false;
    }

}
```

### 数据同步

**app 数据同步**

后台服务（如订单服务）启动时，只将服务名，版本，ip 地址和端口号注册到了 Nacos，并没有实例的权重和启用的插件信息怎么办？

一般在线的实例权重和插件列表都是在管理界面配置，然后动态生效的，所以需要 ship-admin 定时更新实例的权重和插件信息到注册中心。

对应代码 ship-admin 的 NacosSyncListener

```
/**
 * @Author: Ship
 * @Description:
 * @Date: Created in 2020/12/30
 */
@Configuration
public class NacosSyncListener implements ApplicationListener<ContextRefreshedEvent> {

    private static final Logger LOGGER = LoggerFactory.getLogger(NacosSyncListener.class);

    private static ScheduledThreadPoolExecutor scheduledPool = new ScheduledThreadPoolExecutor(1,
            new ShipThreadFactory("nacos-sync", true).create());

    @NacosInjected
    private NamingService namingService;

    @Value("${nacos.discovery.server-addr}")
    private String baseUrl;

    @Resource
    private AppService appService;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (event.getApplicationContext().getParent() != null) {
            return;
        }
        String url = "http://" + baseUrl + NacosConstants.INSTANCE_UPDATE_PATH;
        scheduledPool.scheduleWithFixedDelay(new NacosSyncTask(namingService, url, appService), 0, 30L, TimeUnit.SECONDS);
    }

    class NacosSyncTask implements Runnable {

        private NamingService namingService;

        private String url;

        private AppService appService;

        private Gson gson = new GsonBuilder().create();

        public NacosSyncTask(NamingService namingService, String url, AppService appService) {
            this.namingService = namingService;
            this.url = url;
            this.appService = appService;
        }

        /**
         * Regular update weight,enabled plugins to nacos instance
         */
        @Override
        public void run() {
            try {
                // get all app names
                ListView<String> services = namingService.getServicesOfServer(1, Integer.MAX_VALUE, NacosConstants.APP_GROUP_NAME);
                if (CollectionUtils.isEmpty(services.getData())) {
                    return;
                }
                List<String> appNames = services.getData();
                List<AppInfoDTO> appInfos = appService.getAppInfos(appNames);
                for (AppInfoDTO appInfo : appInfos) {
                    if (CollectionUtils.isEmpty(appInfo.getInstances())) {
                        continue;
                    }
                    for (ServiceInstance instance : appInfo.getInstances()) {
                        Map<String, Object> queryMap = buildQueryMap(appInfo, instance);
                        String resp = OkhttpTool.doPut(url, queryMap, "");
                        LOGGER.debug("response :{}", resp);
                    }
                }

            } catch (Exception e) {
                LOGGER.error("nacos sync task error", e);
            }
        }

        private Map<String, Object> buildQueryMap(AppInfoDTO appInfo, ServiceInstance instance) {
            Map<String, Object> map = new HashMap<>();
            map.put("serviceName", appInfo.getAppName());
            map.put("groupName", NacosConstants.APP_GROUP_NAME);
            map.put("ip", instance.getIp());
            map.put("port", instance.getPort());
            map.put("weight", instance.getWeight().doubleValue());
            NacosMetadata metadata = new NacosMetadata();
            metadata.setAppName(appInfo.getAppName());
            metadata.setVersion(instance.getVersion());
            metadata.setPlugins(String.join(",", appInfo.getEnabledPlugins()));
            map.put("metadata", StringTools.urlEncode(gson.toJson(metadata)));
            map.put("ephemeral", true);
            return map;
        }
    }
}
```

ship-server 再定时从 Nacos 拉取 app 数据更新到本地 Map 缓存。

```
/**
 * @Author: Ship
 * @Description: sync data to local cache
 * @Date: Created in 2020/12/25
 */
@Configuration
public class DataSyncTaskListener implements ApplicationListener<ContextRefreshedEvent> {

    private static ScheduledThreadPoolExecutor scheduledPool = new ScheduledThreadPoolExecutor(1,
            new ShipThreadFactory("service-sync", true).create());

    @NacosInjected
    private NamingService namingService;

    @Autowired
    private ServerConfigProperties properties;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (event.getApplicationContext().getParent() != null) {
            return;
        }
        scheduledPool.scheduleWithFixedDelay(new DataSyncTask(namingService)
                , 0L, properties.getCacheRefreshInterval(), TimeUnit.SECONDS);
        WebsocketSyncCacheServer websocketSyncCacheServer = new WebsocketSyncCacheServer(properties.getWebSocketPort());
        websocketSyncCacheServer.start();
    }


    class DataSyncTask implements Runnable {

        private NamingService namingService;

        public DataSyncTask(NamingService namingService) {
            this.namingService = namingService;
        }

        @Override
        public void run() {
            try {
                // get all app names
                ListView<String> services = namingService.getServicesOfServer(1, Integer.MAX_VALUE, NacosConstants.APP_GROUP_NAME);
                if (CollectionUtils.isEmpty(services.getData())) {
                    return;
                }
                List<String> appNames = services.getData();
                // get all instances
                for (String appName : appNames) {
                    List<Instance> instanceList = namingService.getAllInstances(appName, NacosConstants.APP_GROUP_NAME);
                    if (CollectionUtils.isEmpty(instanceList)) {
                        continue;
                    }
                    ServiceCache.add(appName, buildServiceInstances(instanceList));
                    List<String> pluginNames = getEnabledPlugins(instanceList);
                    PluginCache.add(appName, pluginNames);
                }
                ServiceCache.removeExpired(appNames);
                PluginCache.removeExpired(appNames);

            } catch (NacosException e) {
                e.printStackTrace();
            }
        }

        private List<String> getEnabledPlugins(List<Instance> instanceList) {
            Instance instance = instanceList.get(0);
            Map<String, String> metadata = instance.getMetadata();
            // plugins: DynamicRoute,Auth
            String plugins = metadata.getOrDefault("plugins", ShipPluginEnum.DYNAMIC_ROUTE.getName());
            return Arrays.stream(plugins.split(",")).collect(Collectors.toList());
        }

        private List<ServiceInstance> buildServiceInstances(List<Instance> instanceList) {
            List<ServiceInstance> list = new LinkedList<>();
            instanceList.forEach(instance -> {
                Map<String, String> metadata = instance.getMetadata();
                ServiceInstance serviceInstance = new ServiceInstance();
                serviceInstance.setAppName(metadata.get("appName"));
                serviceInstance.setIp(instance.getIp());
                serviceInstance.setPort(instance.getPort());
                serviceInstance.setVersion(metadata.get("version"));
                serviceInstance.setWeight((int) instance.getWeight());
                list.add(serviceInstance);
            });
            return list;
        }
    }
}
```

**路由规则数据同步**

同时，如果用户在管理后台更新了路由规则，ship-admin 需要推送规则数据到 ship-server，这里参考了 soul 网关的做法利用 websocket 在第一次建立连接后进行全量同步，此后路由规则发生变更就只作增量同步。

服务端 WebsocketSyncCacheServer：

```
/**
 * @Author: Ship
 * @Description:
 * @Date: Created in 2020/12/28
 */
public class WebsocketSyncCacheServer extends WebSocketServer {

    private final static Logger LOGGER = LoggerFactory.getLogger(WebsocketSyncCacheServer.class);

    private Gson gson = new GsonBuilder().create();

    private MessageHandler messageHandler;

    public WebsocketSyncCacheServer(Integer port) {
        super(new InetSocketAddress(port));
        this.messageHandler = new MessageHandler();
    }


    @Override
    public void onOpen(WebSocket webSocket, ClientHandshake clientHandshake) {
        LOGGER.info("server is open");
    }

    @Override
    public void onClose(WebSocket webSocket, int i, String s, boolean b) {
        LOGGER.info("websocket server close...");
    }

    @Override
    public void onMessage(WebSocket webSocket, String message) {
        LOGGER.info("websocket server receive message:\n[{}]", message);
        this.messageHandler.handler(message);
    }

    @Override
    public void onError(WebSocket webSocket, Exception e) {

    }

    @Override
    public void onStart() {
        LOGGER.info("websocket server start...");
    }


    class MessageHandler {

        public void handler(String message) {
            RouteRuleOperationDTO operationDTO = gson.fromJson(message, RouteRuleOperationDTO.class);
            if (CollectionUtils.isEmpty(operationDTO.getRuleList())) {
                return;
            }
            Map<String, List<AppRuleDTO>> map = operationDTO.getRuleList()
                    .stream().collect(Collectors.groupingBy(AppRuleDTO::getAppName));
            if (OperationTypeEnum.INSERT.getCode().equals(operationDTO.getOperationType())
                    || OperationTypeEnum.UPDATE.getCode().equals(operationDTO.getOperationType())) {
                RouteRuleCache.add(map);
            } else if (OperationTypeEnum.DELETE.getCode().equals(operationDTO.getOperationType())) {
                RouteRuleCache.remove(map);
            }
        }
    }
}
```

客户端 WebsocketSyncCacheClient：

```
/**
 * @Author: Ship
 * @Description:
 * @Date: Created in 2020/12/28
 */
@Component
public class WebsocketSyncCacheClient {

    private final static Logger LOGGER = LoggerFactory.getLogger(WebsocketSyncCacheClient.class);

    private WebSocketClient client;

    private RuleService ruleService;

    private Gson gson = new GsonBuilder().create();

    public WebsocketSyncCacheClient(@Value("${ship.server-web-socket-url}") String serverWebSocketUrl,
                                    RuleService ruleService) {
        if (StringUtils.isEmpty(serverWebSocketUrl)) {
            throw new ShipException(ShipExceptionEnum.CONFIG_ERROR);
        }
        this.ruleService = ruleService;
        ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(1,
                new ShipThreadFactory("websocket-connect", true).create());
        try {
            client = new WebSocketClient(new URI(serverWebSocketUrl)) {
                @Override
                public void onOpen(ServerHandshake serverHandshake) {
                    LOGGER.info("client is open");
                    List<AppRuleDTO> list = ruleService.getEnabledRule();
                    String msg = gson.toJson(new RouteRuleOperationDTO(OperationTypeEnum.INSERT, list));
                    send(msg);
                }

                @Override
                public void onMessage(String s) {
                }

                @Override
                public void onClose(int i, String s, boolean b) {
                }

                @Override
                public void onError(Exception e) {
                    LOGGER.error("websocket client error", e);
                }
            };

            client.connectBlocking();
            //使用调度线程池进行断线重连，30秒进行一次
            executor.scheduleAtFixedRate(() -> {
                if (client != null && client.isClosed()) {
                    try {
                        client.reconnectBlocking();
                    } catch (InterruptedException e) {
                        LOGGER.error("reconnect server fail", e);
                    }
                }
            }, 10, 30, TimeUnit.SECONDS);

        } catch (Exception e) {
            LOGGER.error("websocket sync cache exception", e);
            throw new ShipException(e.getMessage());
        }
    }

    public <T> void send(T t) {
        while (!client.getReadyState().equals(ReadyState.OPEN)) {
            LOGGER.debug("connecting ...please wait");
        }
        client.send(gson.toJson(t));
    }
}
```

## 四、测试

### 4.1 动态路由测试

1. 本地启动 nacos ,**sh startup.sh -m standalone**

2. 启动 ship-admin

3. 本地启动两个 ship-example 实例。

   实例 1 配置：

   ```
   ship:
     http:
       app-name: order
       version: gray_1.0
       context-path: /order
       port: 8081
       admin-url: 127.0.0.1:9001
   
   server:
     port: 8081
   
   nacos:
     discovery:
       server-addr: 127.0.0.1:8848
   ```

   实例 2 配置：

   ```
   ship:
     http:
       app-name: order
       version: prod_1.0
       context-path: /order
       port: 8082
       admin-url: 127.0.0.1:9001
   
   server:
     port: 8082
   
   nacos:
     discovery:
       server-addr: 127.0.0.1:8848
   ```

4. 在数据库添加路由规则配置，该规则表示当 http header 中的 name=ship 时请求路由到 gray_1.0 版本的节点。

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)img

1. 启动 ship-server,看到以下日志时则可以进行测试了。

   ```
   2021-01-02 19:57:09.159  INFO 30413 --- [SocketWorker-29] cn.sp.sync.WebsocketSyncCacheServer      : websocket server receive message:
   [{"operationType":"INSERT","ruleList":[{"id":1,"appId":5,"appName":"order","version":"gray_1.0","matchObject":"HEADER","matchKey":"name","matchMethod":1,"matchRule":"ship","priority":50}]}]
   ```

2. 用 Postman 请求http://localhost:9000/order/user/add,POST方式，header设置name=ship，可以看到只有实例1有日志显示。

   ```
   ==========add user,version:gray_1.0
   ```

### 4.2 性能压测

压测环境：