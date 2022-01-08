---
title: shiro框架
date: 2021-04-23 08:04:48
tags:
---

## 源码

### shiro入口

我们打算将 Shiro 放在 Web 应用中使用，只需在 web.xml 中做如下配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
         http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">
 
    <listener>
        <listener-class>org.apache.shiro.web.env.EnvironmentLoaderListener</listener-class>
    </listener>
 
    <filter>
        <filter-name>ShiroFilter</filter-name>
        <filter-class>org.apache.shiro.web.servlet.ShiroFilter</filter-class>
    </filter>
 
    <filter-mapping>
        <filter-name>ShiroFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
 
</web-app>
```

 web.xml 才是整个 Web 应用的核心所在，Web 容器（例如：Tomcat）会提供一些监听器，用于监听 Web 应用的生命周期事件，有两个重要的点可以监听，一个是出生，另一个是死亡，具备这类特性的监听器就是 ServletContextListener。



Shiro 的 EnvironmentLoaderListener 就是一个典型的 ServletContextListener，它也是整个 Shiro Web 应用的入口，不妨先来看看它的静态结构吧：

![img](http://static.oschina.net/uploads/space/2014/0318/162859_TEXh_223750.png)

1. EventListener 是一个标志接口，里面没有任何的方法，Servlet 容器中所有的 Listener 都要继承这个接口<!--这是 Servlet 规范-->

2. ServletContextListener 是一个 ServletContext 的监听器，用于监听容器的启动与关闭事件，包括如下两个方法：
     \- void contextInitialized(ServletContextEvent sce); // 当容器启动时调用
     \- void contextDestroyed(ServletContextEvent sce); // 当容器关闭时调用
     可以从 ServletContextEvent 中直接获取 ServletContext 对象。

3. EnvironmentLoaderListener 不仅实现了 ServletContextListener 接口，也扩展了 EnvironmentLoader 类，是为了在 Servlet 容器中调用 EnvironmentLoader 对象的生命周期方法。

#### EnvironmentLoader

EnvironmentLoaderListener 开始：

```
public class EnvironmentLoaderListener extends EnvironmentLoader implements ServletContextListener {
 
    // 容器启动时调用
    public void contextInitialized(ServletContextEvent sce) {
        initEnvironment(sce.getServletContext());
    }
 
    // 当容器关闭时调用
    public void contextDestroyed(ServletContextEvent sce) {
        destroyEnvironment(sce.getServletContext());
    }
}
```

看来 EnvironmentLoaderListener 只是一个空架子而已，真正干活的人是它“爹”（EnvironmentLoader）：

```java
public class EnvironmentLoader {
 
    // 可在 web.xml 的 context-param 中定义 WebEnvironment 接口的实现类（默认为 IniWebEnvironment）
    public static final String ENVIRONMENT_CLASS_PARAM = "shiroEnvironmentClass";
 
    // 可在 web.xml 的 context-param 中定义 Shiro 配置文件的位置
    public static final String CONFIG_LOCATIONS_PARAM = "shiroConfigLocations";
 
    // 在 ServletContext 中存放 WebEnvironment 的 key
    public static final String ENVIRONMENT_ATTRIBUTE_KEY = EnvironmentLoader.class.getName() + ".ENVIRONMENT_ATTRIBUTE_KEY";
 
    // 从 ServletContext 中获取相关信息，并创建 WebEnvironment 实例
    public WebEnvironment initEnvironment(ServletContext servletContext) throws IllegalStateException {
        // 确保 WebEnvironment 只能创建一次
        if (servletContext.getAttribute(ENVIRONMENT_ATTRIBUTE_KEY) != null) {
            throw new IllegalStateException();
        }
        try {
            // 创建 WebEnvironment 实例
            WebEnvironment environment = createEnvironment(servletContext);
 
            // 将 WebEnvironment 实例放入 ServletContext 中
            servletContext.setAttribute(ENVIRONMENT_ATTRIBUTE_KEY, environment);
            return environment;
        } catch (RuntimeException ex) {
            // 将异常对象放入 ServletContext 中
            servletContext.setAttribute(ENVIRONMENT_ATTRIBUTE_KEY, ex);
            throw ex;
        } catch (Error err) {
            // 将错误对象放入 ServletContext 中
            servletContext.setAttribute(ENVIRONMENT_ATTRIBUTE_KEY, err);
            throw err;
        }
    }
 
    protected WebEnvironment createEnvironment(ServletContext sc) {
        // 确定 WebEnvironment 接口的实现类
        Class<?> clazz = determineWebEnvironmentClass(sc);
 
        // 确保该实现类实现了 MutableWebEnvironment 接口
        if (!MutableWebEnvironment.class.isAssignableFrom(clazz)) {
            throw new ConfigurationException();
        }
 
        // 从 ServletContext 中获取 Shiro 配置文件的位置参数，并判断该参数是否已定义
        String configLocations = sc.getInitParameter(CONFIG_LOCATIONS_PARAM);
        boolean configSpecified = StringUtils.hasText(configLocations);
 
        // 若配置文件位置参数已定义，则需确保该实现类实现了 ResourceConfigurable 接口
        if (configSpecified && !(ResourceConfigurable.class.isAssignableFrom(clazz))) {
            throw new ConfigurationException();
        }
 
        // 通过反射创建 WebEnvironment 实例，将其转型为 MutableWebEnvironment 类型，并将 ServletContext 放入该实例中
        MutableWebEnvironment environment = (MutableWebEnvironment) ClassUtils.newInstance(clazz);
        environment.setServletContext(sc);
 
        // 若配置文件位置参数已定义，且该实例是 ResourceConfigurable 接口的实例（实现了该接口），则将此参数放入该实例中
        if (configSpecified && (environment instanceof ResourceConfigurable)) {
            ((ResourceConfigurable) environment).setConfigLocations(configLocations);
        }
 
        // 可进一步定制 WebEnvironment 实例（在子类中扩展）
        customizeEnvironment(environment);
 
        // 调用 WebEnvironment 实例的 init 方法
        LifecycleUtils.init(environment);
 
        // 返回 WebEnvironment 实例
        return environment;
    }
 
    protected Class<?> determineWebEnvironmentClass(ServletContext servletContext) {
        // 从初始化参数（context-param）中获取 WebEnvironment 接口的实现类
        String className = servletContext.getInitParameter(ENVIRONMENT_CLASS_PARAM);
        // 若该参数已定义，则加载该实现类
        if (className != null) {
            try {
                return ClassUtils.forName(className);
            } catch (UnknownClassException ex) {
                throw new ConfigurationException(ex);
            }
        } else {
            // 否则使用默认的实现类
            return IniWebEnvironment.class;
        }
    }
 
    protected void customizeEnvironment(WebEnvironment environment) {
    }
 
    // 销毁 WebEnvironment 实例
    public void destroyEnvironment(ServletContext servletContext) {
        try {
            // 从 ServletContext 中获取 WebEnvironment 实例
            Object environment = servletContext.getAttribute(ENVIRONMENT_ATTRIBUTE_KEY);
            // 调用 WebEnvironment 实例的 destroy 方法
            LifecycleUtils.destroy(environment);
        } finally {
            // 移除 ServletContext 中存放的 WebEnvironment 实例
            servletContext.removeAttribute(ENVIRONMENT_ATTRIBUTE_KEY);
        }
    }
}
```

看来 EnvironmentLoader 就是为了：

1. 当容器启动时，读取 web.xml 文件，从中获取 WebEnvironment 接口的实现类（默认是 IniWebEnvironment），初始化该实例，并将其加载到 ServletContext 中。

2. 当容器关闭时，销毁 WebEnvironment 实例，并从 ServletContext 将其移除。

这里有两个配置项可以在 web.xml 中进行配置：

```xml
<context-param>
    <param-name>shiroEnvironmentClass</param-name>
    <param-value>WebEnvironment 接口的实现类</param-value>
</context-param>
<context-param>
    <param-name>shiroConfigLocations</param-name>
    <param-value>shiro.ini 配置文件的位置</param-value>
</context-param>
```

在 EnvironmentLoader 中仅用于创建 WebEnvironment 接口的实现类，随后将由这个实现类来加载并解析 shiro.ini 配置文件。

#### WebEnvironment

既然 WebEnvironment 如此重要，那么很有必要了解一下它的静态结构：

![img](http://static.oschina.net/uploads/space/2014/0318/163231_w73G_223750.png)

1. 最底层的 IniWebEnvironment 是 WebEnvironment 接口的默认实现类，它将读取 ini 配置文件，并创建 WebEnvironment 实例。

2. 可以断言，如果需要将 Shiro 配置定义在 XML 或 Properties 配置文件中，那就需要自定义一些WebEnvironment 实现类了。

3. WebEnvironment 的实现类不仅需要实现最顶层的 Environment 接口，还需要实现具有生命周期功能的 Initializable 与 Destroyable 接口。

```java
public class IniWebEnvironment extends ResourceBasedWebEnvironment implements Initializable, Destroyable {
 
    // 默认 shiro.ini 路径
    public static final String DEFAULT_WEB_INI_RESOURCE_PATH = "/WEB-INF/shiro.ini";
 
    // 定义一个 Ini 对象，用于封装 ini 配置项
    private Ini ini;
 
    public Ini getIni() {
        return this.ini;
    }
 
    public void setIni(Ini ini) {
        this.ini = ini;
    }
 
    // 当初始化时调用
    public void init() {
        // 从成员变量中获取 Ini 对象
        Ini ini = getIni();
 
        // 从 web.xml 中获取配置文件位置（在 EnvironmentLoader 中已设置）
        String[] configLocations = getConfigLocations();
 
        // 若成员变量中不存在，则从已定义的配置文件位置获取
        if (CollectionUtils.isEmpty(ini)) {
            ini = getSpecifiedIni(configLocations);
        }
 
        // 若已定义的配置文件中仍然不存在，则从默认的位置获取
        if (CollectionUtils.isEmpty(ini)) {
            ini = getDefaultIni();
        }
 
        // 若还不存在，则抛出异常
        if (CollectionUtils.isEmpty(ini)) {
            throw new ConfigurationException();
        }
 
        // 初始化成员变量
        setIni(ini);
 
        // 解析配置文件，完成初始化工作
        configure();
    }
 
    protected Ini getSpecifiedIni(String[] configLocations) throws ConfigurationException {
        Ini ini = null;
        if (configLocations != null && configLocations.length > 0) {
            // 只能通过第一个配置文件的位置来创建 Ini 对象，且必须有一个配置文件，否则就会报错
            ini = createIni(configLocations[0], true);
        }
        return ini;
    }
 
    protected Ini createIni(String configLocation, boolean required) throws ConfigurationException {
        Ini ini = null;
        if (configLocation != null) {
            // 从指定路径下读取配置文件
            ini = convertPathToIni(configLocation, required);
        }
        if (required && CollectionUtils.isEmpty(ini)) {
            throw new ConfigurationException();
        }
        return ini;
    }
 
    private Ini convertPathToIni(String path, boolean required) {
        Ini ini = null;
        if (StringUtils.hasText(path)) {
            InputStream is = null;
            // 若路径不包括资源前缀（classpath:、url:、file:），则从 ServletContext 中读取，否则从这些资源路径下读取
            if (!ResourceUtils.hasResourcePrefix(path)) {
                is = getServletContextResourceStream(path);
            } else {
                try {
                    is = ResourceUtils.getInputStreamForPath(path);
                } catch (IOException e) {
                    if (required) {
                        throw new ConfigurationException(e);
                    }
                }
            }
            // 将流中的数据加载到 Ini 对象中
            if (is != null) {
                ini = new Ini();
                ini.load(is);
            } else {
                if (required) {
                    throw new ConfigurationException();
                }
            }
        }
        return ini;
    }
 
    private InputStream getServletContextResourceStream(String path) {
        InputStream is = null;
        // 需要将路径进行标准化
        path = WebUtils.normalize(path);
        ServletContext sc = getServletContext();
        if (sc != null) {
            is = sc.getResourceAsStream(path);
        }
        return is;
    }
 
    protected Ini getDefaultIni() {
        Ini ini = null;
        String[] configLocations = getDefaultConfigLocations();
        if (configLocations != null) {
            // 先找到的先使用，后面的无需使用
            for (String location : configLocations) {
                ini = createIni(location, false);
                if (!CollectionUtils.isEmpty(ini)) {
                    break;
                }
            }
        }
        return ini;
    }
 
    protected String[] getDefaultConfigLocations() {
        return new String[]{
            DEFAULT_WEB_INI_RESOURCE_PATH,              // /WEB-INF/shiro.ini
            IniFactorySupport.DEFAULT_INI_RESOURCE_PATH // classpath:shiro.ini
        };
    }
 
    protected void configure() {
        // 清空这个 Bean 容器（一个 Map<String, Object> 对象，在 DefaultEnvironment 中定义）
        this.objects.clear();
 
        // 创建基于 Web 的 SecurityManager 对象（WebSecurityManager）
        WebSecurityManager securityManager = createWebSecurityManager();
        setWebSecurityManager(securityManager);
 
        // 初始化 Filter Chain 解析器（用于解析 Filter 规则）
        FilterChainResolver resolver = createFilterChainResolver();
        if (resolver != null) {
            setFilterChainResolver(resolver);
        }
    }
 
    protected WebSecurityManager createWebSecurityManager() {
        // 通过工厂对象来创建 WebSecurityManager 实例
        WebIniSecurityManagerFactory factory;
        Ini ini = getIni();
        if (CollectionUtils.isEmpty(ini)) {
            factory = new WebIniSecurityManagerFactory();
        } else {
            factory = new WebIniSecurityManagerFactory(ini);
        }
        WebSecurityManager wsm = (WebSecurityManager) factory.getInstance();
 
        // 从工厂中获取 Bean Map 并将其放入 Bean 容器中
        Map<String, ?> beans = factory.getBeans();
        if (!CollectionUtils.isEmpty(beans)) {
            this.objects.putAll(beans);
        }
 
        return wsm;
    }
 
    protected FilterChainResolver createFilterChainResolver() {
        FilterChainResolver resolver = null;
        Ini ini = getIni();
        if (!CollectionUtils.isEmpty(ini)) {
            // Filter 可以从 [urls] 或 [filters] 片段中读取
            Ini.Section urls = ini.getSection(IniFilterChainResolverFactory.URLS);
            Ini.Section filters = ini.getSection(IniFilterChainResolverFactory.FILTERS);
            if (!CollectionUtils.isEmpty(urls) || !CollectionUtils.isEmpty(filters)) {
                // 通过工厂对象创建 FilterChainResolver 实例
                IniFilterChainResolverFactory factory = new IniFilterChainResolverFactory(ini, this.objects);
                resolver = factory.getInstance();
            }
        }
        return resolver;
    }
}
```

看来 IniWebEnvironment 就是为了：

1. 查找并加载 shiro.ini 配置文件，首先从自身成员变量里查找，然后从 web.xml 中查找，然后从 /WEB-INF 下查找，然后从 classpath 下查找，若均未找到，则直接报错。

2. 当找到了 ini 配置文件后就开始解析，此时构造了一个 Bean 容器（相当于一个轻量级的 IOC 容器），最终的目标是为了创建 WebSecurityManager 对象与 FilterChainResolver 对象，创建过程使用了 Abstract Factory 模式

####  Abstract Factory 

![img](http://static.oschina.net/uploads/space/2014/0318/163354_8Zjs_223750.png)

其中有两个 Factory 需要关注：
\- WebIniSecurityManagerFactory 用于创建 WebSecurityManager。
\- IniFilterChainResolverFactory 用于创建 FilterChainResolver。

通过以上分析，相信 EnvironmentLoaderListener 已经不再神秘了，无非就是在容器启动时创建 WebEnvironment 对象，并由该对象来读取 Shiro 配置文件，创建WebSecurityManager 与 FilterChainResolver 对象，它们都在后面将要出现的 ShiroFilter 中起到了重要作用。

从 web.xml 中同样可以得知，ShiroFilter 是整个 Shiro 框架的门面，因为它拦截了所有的请求，后面是需要 Authentication（认证）还是需要 Authorization（授权）都由它说了算。



### shiro filter

在上一篇中，我们分析了 Shiro Web 应用的入口 —— EnvironmentLoaderListener，它是一个 ServletContextListener，在 Web 容器启动的时候，它为我们创建了两个非常重要的对象：

- WebSecurityManager：它是用于 Web 环境的 SecurityManager 对象，通过读取 shiro.ini 中 [main] 片段生成的，我们可以通过 SecurityUtils.getSecurityManager 方法获取该对象。
- FilterChainResolver：它是 shiro.ini 中 [urls] 片段所配置的 Filter Chain 的解析器，可对一个 URL 配置一个或多个 Filter（用逗号分隔），Shiro 也为我们提供了几个默认的 Filter。

 Shiro Web 的第二个核心对象 —— ShiroFilter，它是在整个 Shiro Web 应用中请求的门户，也就是说，所有的请求都会被 ShiroFilter 拦截并进行相应的链式处理。

![img](http://static.oschina.net/uploads/space/2014/0321/154813_mpy9_223750.png)

#### filter接口

上图可见，ShiroFilter 往上竟然有五层，最上层是 Filter（即 javax.servlet.Filter），它是 Servlet 规范中的 Filter 接口，代码如下：<!--只有实现的sevlet的filter接口，才能被web容器识别和加载为filter-->

```
public interface Filter {

    void init(FilterConfig filterConfig) throws ServletException;

    void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException;

    void destroy();
}
```

Filter 接口中的三个方法分别在 Filter 生命周期的三个时期内由 Web 容器来调用，分别是：初始化、执行、销毁。

但与 Filter 接口同一级别下竟然还有一个名为 ServletContextSupport 的类，它又是起什么作用的呢？

#### 封装ServletContext

打开 ServletContextSupport 的源码便知，它是 Shiro 为了封装 ServletContext 的而提供的一个类，代码如下：

```
/**
 * 封装 ServletContext
 */
public class ServletContextSupport {

    private ServletContext servletContext;

    public ServletContext getServletContext() {
        return servletContext;
    }

    public void setServletContext(ServletContext servletContext) {
        this.servletContext = servletContext;
    }

    @SuppressWarnings({"UnusedDeclaration"})
    protected String getContextInitParam(String paramName) {
        return getServletContext().getInitParameter(paramName);
    }

    private ServletContext getRequiredServletContext() {
        ServletContext servletContext = getServletContext();
        if (servletContext == null) {
            throw new IllegalStateException();
        }
        return servletContext;
    }

    @SuppressWarnings({"UnusedDeclaration"})
    protected void setContextAttribute(String key, Object value) {
        if (value == null) {
            removeContextAttribute(key);
        } else {
            getRequiredServletContext().setAttribute(key, value);
        }
    }

    @SuppressWarnings({"UnusedDeclaration"})
    protected Object getContextAttribute(String key) {
        return getRequiredServletContext().getAttribute(key);
    }

    protected void removeContextAttribute(String key) {
        getRequiredServletContext().removeAttribute(key);
    }

    @Override
    public String toString() {
        return toStringBuilder().toString();
    }

    protected StringBuilder toStringBuilder() {
        return new StringBuilder(super.toString());
    }
}
```



通过这个类，我们可以方便的操纵 ServletContext 对象（使用其中的属性），那么这个 ServletContext 对象又是如何来初始化的呢？

不妨看看 Filter 与 ServletContextSupport 的子类 AbstractFilter 吧，代码如下：

```
/**
 * 初始化 ServletContext 并封装 FilterConfig
 */
public abstract class AbstractFilter extends ServletContextSupport implements Filter {

    protected FilterConfig filterConfig;

    public FilterConfig getFilterConfig() {
        return filterConfig;
    }

    public void setFilterConfig(FilterConfig filterConfig) {
        // 初始化 FilterConfig 与 ServletContext
        this.filterConfig = filterConfig;
        setServletContext(filterConfig.getServletContext());
    }

    protected String getInitParam(String paramName) {
        // 从 FilterConfig 中获取初始参数
        FilterConfig config = getFilterConfig();
        if (config != null) {
            return StringUtils.clean(config.getInitParameter(paramName));
        }
        return null;
    }

    public final void init(FilterConfig filterConfig) throws ServletException {
        // 初始化 FilterConfig
        setFilterConfig(filterConfig);
        try {
            // 在子类中实现该模板方法
            onFilterConfigSet();
        } catch (Exception e) {
            if (e instanceof ServletException) {
                throw (ServletException) e;
            } else {
                throw new ServletException(e);
            }
        }
    }

    protected void onFilterConfigSet() throws Exception {
    }

    public void destroy() {
    }
}
```



看到这个类的第一感觉就是，它对 FilterConfig 进行了封装，为什么要封装 FilterConfig 呢？就是想通过它来获取 ServletContext。可见，在 init 方法中完成了 FilterConfig 的初始化，并提供了一个名为 onFilterConfigSet 的模板方法，让它的子类去实现其中的细节。

在阅读 AbstractFilter 的子类 NameableFilter 的源码之前，不妨先看看 NameableFilter 实现了一个很有意思的接口 Nameable，代码如下：

```
/**
 * 确保实现该接口的类可进行命名（具有唯一的名称）
 */
public interface Nameable {

    void setName(String name);
}
```



仅提供了一个 setName 的方法，目的就是为了让其子类能够提供一个唯一的 Filter Name，如果子类不提供怎么办呢？

相信 Nameable 的实现类也就是 AbstractFilter 的子类 NameableFilter 会告诉我们想要的答案，代码如下：

```
/**
 * 提供 Filter Name 的 get/set 方法
 */
public abstract class NameableFilter extends AbstractFilter implements Nameable {

    private String name;

    protected String getName() {
        // 若成员变量 name 为空，则从 FilterConfig 中获取 Filter Name
        if (this.name == null) {
            FilterConfig config = getFilterConfig();
            if (config != null) {
                this.name = config.getFilterName();
            }
        }
        return this.name;
    }

    public void setName(String name) {
        this.name = name;
    }

    protected StringBuilder toStringBuilder() {
        String name = getName();
        if (name == null) {
            return super.toStringBuilder();
        } else {
            StringBuilder sb = new StringBuilder();
            sb.append(name);
            return sb;
        }
    }
}
```



看到了 NameableFilter 中的 getName 方法，我们应该清楚了，每个 Filter 必须有一个名字，可通过 setName 方法设置的，如果不设置就取该 Filter 默认的名字，也就是在 web.xml 中配置的 filter-name 了。此外，这里还通过一个 toStringBuilder 方法完成了类似 toString 方法，不过暂时还没什么用途，可能以后会有用。

以上这一切都是为了让每个 Filter 有一个名字，而且这个名字最好是唯一的（这一点在 Shiro 源码中没有得到控制）。此外，在 shiro.ini 的 [urls] 片段的配置满足一定规则的，例如：

```
[urls]
/foo = ssl, authc
```



等号左边的是 URL，右边的是 Filter Chian，一个或多个 Filter，每个 Filter 用逗号进行分隔。

对于 /foo 这个 URL 而言，可先后通过 ssl 与 authc 这两个 Filter。如果我们同时配置了两个 ssl，这个 URL 会被 ssl 拦截两次吗？答案是否定的，因为 Shiro 为我们提供了一个“一次性 Filter”的原则，也就是保证了每个请求只能被同一个 Filter 拦截一次，而且仅此一次。

这样的机制是如何实现的呢？我们不妨看看 NameableFilter 的子类 OncePerRequestFilter 吧，代码如下：

```
/**
 * 确保每个请求只能被 Filter 过滤一次
 */
public abstract class OncePerRequestFilter extends NameableFilter {

    // 已过滤属性的后缀名
    public static final String ALREADY_FILTERED_SUFFIX = ".FILTERED";

    // 是否开启过滤功能
    private boolean enabled = true;

    public boolean isEnabled() {
        return enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }

    public final void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 获取 Filter 已过滤的属性名
        String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
        // 判断是否已过滤
        if (request.getAttribute(alreadyFilteredAttributeName) != null) {
            // 若已过滤，则进入 FilterChain 中下一个 Filter
            filterChain.doFilter(request, response);
        } else {
            // 若未过滤，则判断是否未开启过滤功能（其中 shouldNotFilter 方法将被废弃，由 isEnabled 方法取代）
            if (!isEnabled(request, response) || shouldNotFilter(request)) {
                // 若未开启，则进入 FilterChain 中下一个 Filter
                filterChain.doFilter(request, response);
            } else {
                // 若已开启，则将已过滤属性设置为 true（只要保证 Request 中有这个属性即可）
                request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
                try {
                    // 在子类中执行具体的过滤操作
                    doFilterInternal(request, response, filterChain);
                } finally {
                    // 当前 Filter 执行结束需移除 Request 中的已过滤属性
                    request.removeAttribute(alreadyFilteredAttributeName);
                }
            }
        }
    }

    protected String getAlreadyFilteredAttributeName() {
        String name = getName();
        if (name == null) {
            name = getClass().getName();
        }
        return name + ALREADY_FILTERED_SUFFIX;
    }

    @SuppressWarnings({"UnusedParameters"})
    protected boolean isEnabled(ServletRequest request, ServletResponse response) throws ServletException, IOException {
        return isEnabled();
    }

    @Deprecated
    @SuppressWarnings({"UnusedDeclaration"})
    protected boolean shouldNotFilter(ServletRequest request) throws ServletException {
        return false;
    }

    protected abstract void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException;
}
```



如何确保每个请求只会被同一个 Filter 拦截一次呢？Shiro 提供了一个超简单的解决方案：在 Requet 中放置一个后缀为 .FILTERED 的属性，在执行具体拦截操作（即 doFilterInternal 方法）之前放入该属性，执行完毕后移除该属性。

在 Shiro 的 Filter Chian 配置中，如果我们想禁用某个 Filter，如何实现呢？OncePerRequestFilter 也为我们提供了一个 enabled 的属性，方便我们可以在 shiro.ini 中随时禁用某个 Filter，例如：

```
[main]
ssl.enabled = false

[urls]
/foo = ssl, authc
```



这样一来 ssl 这个 Filter 就被我们给禁用了，以后想开启 ssl 的话，完全不需要在 urls 配置中一个个手工来添加，只需把 ssl.enabled 设置为 true，或注释掉该行，或直接删除该行即可。

可见，OncePerRequestFilter 给我们提供了一个模板方法 doFilterInternal，在其子类中我们需要实现该方法的具体细节，那么谁来实现呢？不妨继续看下面的 AbstractShiroFilter 吧，代码如下：

```
/**
 * 确保可通过 SecurityUtils 获取 SecurityManager，并执行过滤器操作
 */
public abstract class AbstractShiroFilter extends OncePerRequestFilter {

    // 是否可以通过 SecurityUtils 获取 SecurityManager
    private static final String STATIC_INIT_PARAM_NAME = "staticSecurityManagerEnabled";

    private WebSecurityManager securityManager;
    private FilterChainResolver filterChainResolver;
    private boolean staticSecurityManagerEnabled;

    protected AbstractShiroFilter() {
        this.staticSecurityManagerEnabled = false;
    }

    public WebSecurityManager getSecurityManager() {
        return securityManager;
    }

    public void setSecurityManager(WebSecurityManager sm) {
        this.securityManager = sm;
    }

    public FilterChainResolver getFilterChainResolver() {
        return filterChainResolver;
    }

    public void setFilterChainResolver(FilterChainResolver filterChainResolver) {
        this.filterChainResolver = filterChainResolver;
    }

    public boolean isStaticSecurityManagerEnabled() {
        return staticSecurityManagerEnabled;
    }

    public void setStaticSecurityManagerEnabled(boolean staticSecurityManagerEnabled) {
        this.staticSecurityManagerEnabled = staticSecurityManagerEnabled;
    }

    // 这是 AbstractFilter 提供的在 init 时需要执行的方法
    protected final void onFilterConfigSet() throws Exception {
        // 从 web.xml 中读取 staticSecurityManagerEnabled 参数（默认为 false）
        applyStaticSecurityManagerEnabledConfig();
        // 初始化（在子类中实现）
        init();
        // 确保 SecurityManager 必须存在
        ensureSecurityManager();
        // 若已开启 static 标志，则将当前的 SecurityManager 放入 SecurityUtils 中，以后可以随时获取
        if (isStaticSecurityManagerEnabled()) {
            SecurityUtils.setSecurityManager(getSecurityManager());
        }
    }

    private void applyStaticSecurityManagerEnabledConfig() {
        String value = getInitParam(STATIC_INIT_PARAM_NAME);
        if (value != null) {
            Boolean b = Boolean.valueOf(value);
            if (b != null) {
                setStaticSecurityManagerEnabled(b);
            }
        }
    }

    public void init() throws Exception {
    }

    private void ensureSecurityManager() {
        // 首先获取当前的 SecurityManager，若不存在，则创建默认的 SecurityManager（即 DefaultWebSecurityManager）
        WebSecurityManager securityManager = getSecurityManager();
        if (securityManager == null) {
            securityManager = createDefaultSecurityManager();
            setSecurityManager(securityManager);
        }
    }

    protected WebSecurityManager createDefaultSecurityManager() {
        return new DefaultWebSecurityManager();
    }

    // 这是 OncePerRequestFilter 提供的在 doFilter 时需要执行的方法
    protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain) throws ServletException, IOException {
        Throwable t = null;
        try {
            // 返回被 Shiro 包装过的 Request 与 Response 对象
            final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
            final ServletResponse response = prepareServletResponse(request, servletResponse, chain);
            // 创建 Shiro 的 Subject 对象
            final Subject subject = createSubject(request, response);
            // 使用异步的方式执行相关操作
            subject.execute(new Callable() {
                public Object call() throws Exception {
                    // 更新 Session 的最后访问时间
                    updateSessionLastAccessTime(request, response);
                    // 执行 Shiro 的 Filter Chain
                    executeChain(request, response, chain);
                    return null;
                }
            });
        } catch (ExecutionException ex) {
            t = ex.getCause();
        } catch (Throwable throwable) {
            t = throwable;
        }
        if (t != null) {
            if (t instanceof ServletException) {
                throw (ServletException) t;
            }
            if (t instanceof IOException) {
                throw (IOException) t;
            }
            throw new ServletException(t);
        }
    }

    @SuppressWarnings({"UnusedDeclaration"})
    protected ServletRequest prepareServletRequest(ServletRequest request, ServletResponse response, FilterChain chain) {
        ServletRequest toUse = request;
        if (request instanceof HttpServletRequest) {
            // 获取包装后的 Request 对象（使用 ShiroHttpServletRequest 进行包装）
            HttpServletRequest http = (HttpServletRequest) request;
            toUse = wrapServletRequest(http);
        }
        return toUse;
    }

    protected ServletRequest wrapServletRequest(HttpServletRequest orig) {
        return new ShiroHttpServletRequest(orig, getServletContext(), isHttpSessions());
    }

    protected boolean isHttpSessions() {
        return getSecurityManager().isHttpSessionMode();
    }

    @SuppressWarnings({"UnusedDeclaration"})
    protected ServletResponse prepareServletResponse(ServletRequest request, ServletResponse response, FilterChain chain) {
        ServletResponse toUse = response;
        if (!isHttpSessions() && (request instanceof ShiroHttpServletRequest) && (response instanceof HttpServletResponse)) {
            // 获取包装后的 Response 对象（使用 ShiroHttpServletResponse 进行包装）
            toUse = wrapServletResponse((HttpServletResponse) response, (ShiroHttpServletRequest) request);
        }
        return toUse;
    }

    protected ServletResponse wrapServletResponse(HttpServletResponse orig, ShiroHttpServletRequest request) {
        return new ShiroHttpServletResponse(orig, getServletContext(), request);
    }

    protected WebSubject createSubject(ServletRequest request, ServletResponse response) {
        return new WebSubject.Builder(getSecurityManager(), request, response).buildWebSubject();
    }

    @SuppressWarnings({"UnusedDeclaration"})
    protected void updateSessionLastAccessTime(ServletRequest request, ServletResponse response) {
        // 仅对本地 Session 做如下操作
        if (!isHttpSessions()) {
            // 获取 Subject（实际上是从 ThreadLocal 中获取的）
            Subject subject = SecurityUtils.getSubject();
            if (subject != null) {
                // 从 Subject 中获取 Session
                Session session = subject.getSession(false);
                if (session != null) {
                    // 更新 Session 对象的 lastAccessTime 属性
                    session.touch();
                }
            }
        }
    }

    protected void executeChain(ServletRequest request, ServletResponse response, FilterChain origChain) throws IOException, ServletException {
        // 获取 Shiro 代理后的 FilterChain 对象，并进行链式处理
        FilterChain chain = getExecutionChain(request, response, origChain);
        chain.doFilter(request, response);
    }

    protected FilterChain getExecutionChain(ServletRequest request, ServletResponse response, FilterChain origChain) {
        FilterChain chain = origChain;
        // 获取 FilterChainResolver，若不存在，则返回原始的 FilterChain
        FilterChainResolver resolver = getFilterChainResolver();
        if (resolver == null) {
            return origChain;
        }
        // 通过 FilterChainResolver 获取 ProxiedFilterChain
        FilterChain resolved = resolver.getChain(request, response, origChain);
        if (resolved != null) {
            chain = resolved;
        }
        return chain;
    }
}
```



这个 AbstractShiroFilter 类代码稍微有点长，因为它干了许多的事情，主要实现了两个模板方法：onFilterConfigSet 与 doFilterInternal，以上代码中均已对它们做了详细的注释。

其中，在 onFilterConfigSet 中实际上提供了一个框架，只是将 SecurityManager 放入 SecurityUtils 这个工具类中，至于具体行为还是放在子类的 init 方法中去实现，而这个子类就是 ShiroFilter，代码如下：



```
/**
 * 初始化过滤器
 */
public class ShiroFilter extends AbstractShiroFilter {

    @Override
    public void init() throws Exception {
        // 从 ServletContext 中获取 WebEnvironment（该对象已通过 EnvironmentLoader 创建）
        WebEnvironment env = WebUtils.getRequiredWebEnvironment(getServletContext());

        // 将 WebEnvironment 中的 WebSecurityManager 放入 AbstractShiroFilter 中
        setSecurityManager(env.getWebSecurityManager());

        // 将 WebEnvironment 中的 FilterChainResolver 放入 AbstractShiroFilter 中
        FilterChainResolver resolver = env.getFilterChainResolver();
        if (resolver != null) {
            setFilterChainResolver(resolver);
        }
    }
}
```



在 ShiroFilter 中只用做初始化的行为，就是从 WebEnvironment 中分别获取 WebSecurityManager 与 FilterChainResolver，其它的事情都由它的父类去实现了。

到此为止，ShiroFilter 的源码已基本分析完毕，当然还有些非常有意思的代码，这里没有进行分析，例如：

- 通过 ShiroHttpServletRequest 来包装 Request
- 通过 ShiroHttpServletResponse 来包装 Response
- 通过 Session 来代理 HttpSession
- 提供 FilterChain 的代理机制
- 使用 ThreadContext 来保证线程安全

这些有意思的代码，我就不继续分析了，留点滋味让大家去慢慢品尝吧！

最后需要补充说明的是，Shiro 的 Filter 架构体系是非常庞大的，这里仅对 ShiroFilter 进行了分析，整个 Filter 静态结构看起来是这样的：

![img](http://static.oschina.net/uploads/space/2014/0321/172903_VINL_223750.png)

可见，在 OncePerRequestFilter 下有两个分支，本文只分析了 ShiroFilter 这个分支，另外还有一个 AdviceFilter 分支，它提供了 AOP 功能的 Filter，这些 Filter 就是 Shiro 为我们提供的默认 Filter：

| 名称              | 类名                                                         |
| ----------------- | ------------------------------------------------------------ |
| anon              | [org.apache.shiro.web.filter.authc.AnonymousFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthc%2FAnonymousFilter.html) |
| authc             | [org.apache.shiro.web.filter.authc.FormAuthenticationFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthc%2FFormAuthenticationFilter.html) |
| authcBasic        | [org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthc%2FBasicHttpAuthenticationFilter.html) |
| logout            | [org.apache.shiro.web.filter.authc.LogoutFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthc%2FLogoutFilter.html) |
| noSessionCreation | [org.apache.shiro.web.filter.session.NoSessionCreationFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fsession%2FNoSessionCreationFilter.html) |
| perms             | [org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthz%2FPermissionsAuthorizationFilter.html) |
| port              | [org.apache.shiro.web.filter.authz.PortFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthz%2FPortFilter.html) |
| rest              | [org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthz%2FHttpMethodPermissionFilter.html) |
| roles             | [org.apache.shiro.web.filter.authz.RolesAuthorizationFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthz%2FRolesAuthorizationFilter.html) |
| ssl               | [org.apache.shiro.web.filter.authz.SslFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthz%2FSslFilter.html) |
| user              | [org.apache.shiro.web.filter.authc.UserFilter](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Fshiro.apache.org%2Fstatic%2Fcurrent%2Fapidocs%2Forg%2Fapache%2Fshiro%2Fweb%2Ffilter%2Fauthc%2FUserFilter.html) |



## 应用

### 前后端分离

to determine the identity of the user is through the sessionId, and the sessionId is stored in the cookie, we can return the sessionId as the response header to the front end after login, each request is accompanied by this information, then we customize a SessionManager, overwrite His getSessionId method can be:
Return the sessionId to the frontend after logging in:

```javascript
@PostMapping("login")
@Responebody
    public Result login(String username, String password, HttpServletResponse response){
        Result result = null;
        UsernamePasswordToken token = new UsernamePasswordToken(username,password);
        Subject subject = SecurityUtils.getSubject();
        try {
            subject.login(token);
            Serializable id = subject.getSession().getId();
            response.setHeader("token",id.toString());/ / Get the state of the sessionId, return to the front end
            result = ResultGenerator.getSuucessResult();
        }catch (Exception e){
            result = ResultGenerator.getFailResult(e.getMessage());
        }
        return result;
    }

```

Custom sessionManager, the way to get the sessionId is no longer a heavy cookie but in the requsetHeader:

```javascript
public class MyWebSessionManager extends DefaultWebSessionManager {
    @Override
    protected Serializable getSessionId(ServletRequest request, ServletResponse response) {
        HttpServletRequest httpServletRequest = WebUtils.toHttp(request);
        String token = httpServletRequest.getHeader("token");
        System.out.println("token："+token);
        request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_SOURCE, "token");
        request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID, token);
        request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_IS_VALID, Boolean.TRUE);
        return token;
    }

}
```

Set to use the change sessionManager, generic can also use ioc injection without new:

```javascript
@Bean
    public DefaultWebSecurityManager webSecurityManager(MyRealm myRealm){
        DefaultWebSecurityManager webSecurityManager = new DefaultWebSecurityManager();
        MyWebSessionManager webSessionManager = new MyWebSessionManager();/ / Use a custom sessionManager
        webSessionManager.setGlobalSessionTimeout(360000l);/ / Set the session expiration time 1 hour
        webSecurityManager.setSessionManager(webSessionManager);
        webSecurityManager.setRealm(myRealm);
        return webSecurityManager;
    }
```

With such a simple configuration, most of the rights management of shiro can be realized by separating the front and back ends.