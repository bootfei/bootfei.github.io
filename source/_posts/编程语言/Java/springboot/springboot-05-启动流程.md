---
title: springboot-04-父子容器
date: 2021-04-16 09:30:13
tags: [java, springboot]
---

由于该系统是底层系统，以微服务形式对外暴露dubbo服务，所以本流程中SpringBoot不基于jetty或者tomcat等容器启动方式发布服务，而是以执行程序方式启动来发布

# SpringBoot 启动过程

![img](https://upload-images.jianshu.io/upload_images/6912735-51aa162747fcdc3d.png)

启动流程主要分为三个部分，

第一部分进行 `SpringApplication` 的初始化模块，配置一些基本的**环境变量、资源、构造器、监听器**。

第二部分实现了应用具体的启动方案，包括启动流程的监听模块、加载配置环境模块、及核心的创建上下文环境模块

第三部分是自动化配置模块，该模块作为springboot自动配置核心，在后面的分析中会详细讨论。在下面的启动程序中我们会串联起结构中的主要功能。



## 第一部分：初始化

### @SpringApplication注解

 每个SpringBoot程序都有一个主入口，也就是main方法，main里面调用SpringApplication.run()启动整个spring-boot程序。SpringBoot要求该main方法所在类必须使用@SpringBootApplication注解，以及@ImportResource注解(if need)

@SpringBootApplication包括三个注解：

| 注解                     | 功能                                                         |
| ------------------------ | ------------------------------------------------------------ |
| @EnableAutoConfiguration | 自动配置：从 `classpath` 中搜寻所有的 `META-INF/spring.factories` 配置文件，并将其中 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 对应的配置项通过**反射实例化**为对应的标注了 `@Configuration` 的 `JavaConfig` 形式的 IoC 容器配置类，然后汇总为一个并加载到 IoC 容器。 |
| @SpringBootConfiguration | 源码内部其实是`@Configuration`：被标注的类等于在spring的XML配置文件中`applicationContext.xml`，装配所有bean事务，提供了一个spring的上下文环境Context |
| @ComponentScan           | 组件扫描，可自动发现和装配Bean，默认扫描SpringApplication的run(xxx.class)中, xxx.class所在的包路径下文件，所以最好将该启动类放到根包路径下。`@ComponentScan`通常与`@Configuration`一起配合使用，相当于xml里面的`<context:component-scan>`，用来告诉Spring需要扫描哪些包或类。如果不设值的话默认扫描@ComponentScan注解所在类的同级类和同级目录下的所有类  , *所以对于一个Spring Boot项目，一般会把入口类放在顶层目录中，这样就能够保证源码目录下的所有类都能够被扫描到*。 |



### SpringApplication#run(xxx.class, args)方法

```java
// SpringApplication.class
// 构造实例，在2nd step中被调用
public SpringApplication(Object... sources) {
    initialize(sources);
}

//在构造函数中被调用
private void initialize(Object[] sources) {
    if (sources != null && sources.length > 0) {
        this.sources.addAll(Arrays.asList(sources));
    }
    // 推断是否为web环境
    this.webEnvironment = deduceWebEnvironment();
    // 设置初始化器
    setInitializers((Collection) getSpringFactoriesInstances(
        ApplicationContextInitializer.class));
    // 设置监听器
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 推断应用入口类
    this.mainApplicationClass = deduceMainApplicationClass();
}

//1st step
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
  return run(new Class[]{primarySource}, args);
}

//2nd step
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
  //我们可以发现其构造方法在这，其实调用了一个初始化的initialize方法
  return (new SpringApplication(primarySources)).run(args);
}

//3rd step
public ConfigurableApplicationContext run(String... args) {
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();
  ConfigurableApplicationContext context = null;
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
  this.configureHeadlessProperty();
  //1.创建了应用的监听器SpringApplicationRunListeners并开始监听(其实是在构造函数中完成的)
  SpringApplicationRunListeners listeners = this.getRunListeners(args);
  listeners.starting();

  Collection exceptionReporters;
  try {
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    //2.加载SpringBoot配置环境(ConfigurableEnvironment)
    ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
    this.configureIgnoreBeanInfo(environment);
    Banner printedBanner = this.printBanner(environment);
    context = this.createApplicationContext();
    exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
    //5.prepareContext()方法将listeners、environment、applicationArguments、banner等重要组件与上下文对象关联
    this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    //6.实现spring-boot-starter-(mybatis、redis等)自动化配置的关键，包括spring.factories的加载，bean的实例化等核心工作
    this.refreshContext(context);
    this.afterRefresh(context, applicationArguments);
    stopWatch.stop();
    if (this.logStartupInfo) {
      (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
    }

    listeners.started(context);
    this.callRunners(context, applicationArguments);
  } catch (Throwable var10) {
    this.handleRunFailure(context, var10, exceptionReporters, listeners);
    throw new IllegalStateException(var10);
  }

  try {
    listeners.running(context);
    return context;
  } catch (Throwable var9) {
    this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
    throw new IllegalStateException(var9);
  }
}
```



run() 方法中实现了如下几个关键步骤：

1.创建了应用的监听器`SpringApplicationRunListeners`并开始监听

2.加载SpringBoot配置环境(ConfigurableEnvironment)，如果是通过web容器发布，会加载`StandardEnvironment`，其最终也是继承了`ConfigurableEnvironment`，`变量environment对象`最终都实现了`PropertyResolver`接口，所以我们平时通过`变量environment对象`获取配置文件中指定Key对应的value时，就是调用了`propertyResolver`接口的`getProperty()`方法

3.配置`变量environment对象`加入到监听器对象中`SpringApplicationRunListeners`

4.创建`应用配置上下文ConfigurableApplicationContext`(即run方法的返回对象)，方法会先获取显式设置的应用上下文(applicationContextClass)，如果不存在，再加载默认的环境配置（通过是否是web environment判断），默认选择AnnotationConfigApplicationContext注解上下文（通过扫描所有注解类来加载bean），最后通过BeanUtils实例化上下文对象，并返回，ConfigurableApplicationContext类图如下：

![img](https:////upload-images.jianshu.io/upload_images/6912735-797f3d2c57b625bc.png)

主要看其继承的两个方向：

LifeCycle：生命周期类，定义了start启动、stop结束、isRunning是否运行中等生命周期空值方法

ApplicationContext：应用上下文类，其主要继承了beanFactory(bean的工厂类)

5.回到run方法内，`prepareContext()`方法将listeners、environment、applicationArguments、banner等重要组件与上下文对象关联

6.接下来的refreshContext(context)方法(初始化方法如下)将是实现spring-boot-starter-(mybatis、redis等)自动化配置的关键，包括spring.factories的加载，bean的实例化等核心工作。



配置结束后，Springboot做了一些基本的收尾工作，返回了应用环境上下文。回顾整体流程，Springboot的启动，主要创建了配置环境(environment)、事件监听(listeners)、应用上下文(applicationContext)，并基于以上条件，在容器中开始实例化我们需要的Bean，至此，通过SpringBoot启动的程序已经构造完成，接下来我们来探讨自动化配置是如何实现。

### 加载 ApplicationContextInitializer

加载所有配置的 `ApplicationContextInitializer` 并进行实例化，加载 `ApplicationContextInitializer` 是在 `SpringFactoriesLoader.loadFactoryNames` 方法里面进行的。这个方法会尝试从类路径的 `META-INF/spring.factories` 读取相应配置文件，然后进行遍历，读取配置文件中Key为：`org.springframework.context.ApplicationContextInitializer` 的 value。以 spring-boot 这个包为例，它的 `META-INF/spring.factories` 部分定义如下所示：

```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.context.embedded.ServerPortInfoApplicationContextInitializer
```

> 接口 `ApplicationContextInitializer` 的定义

```Java
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

    /**
     * Initialize the given application context.
     * @param applicationContext the application to configure
     */
    void initialize(C applicationContext);

}
```

### 加载 ApplicationListener

以 spring-boot 这个包中的 `spring.factories` 为例，看看相应的 Key-Value ：

```
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener,\
org.springframework.boot.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.logging.LoggingApplicationListener
```

> 接口 `ApplicationListener` 的定义

```Java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

    /**
     * Handle an application event.
     * @param event the event to respond to
     */
    void onApplicationEvent(E event);

}
```

## 第二部分：开始启动

### 加载 SpringApplicationRunListener

从 `META-INF/spring.factories` 中读取 Key 为 `org.springframework.boot.SpringApplicationRunListener 的Values`：

比如在 spring-boot 包中的定义的 `spring.factories`:

```
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

### 准备 Environment

```Java
protected void configureEnvironment(ConfigurableEnvironment environment,
        String[] args) {
    configurePropertySources(environment, args);
    configureProfiles(environment, args);
}
```

### 创建 Context

```Java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            contextClass = Class.forName(this.webEnvironment
                    ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, "
                            + "please specify an ApplicationContextClass",
                    ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
}

// WEB应用的上下文类型
public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext";

// 非WEB应用的上下文类型
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
            + "annotation.AnnotationConfigApplicationContext";
```

## 参考

- [SpringBoot启动流程解析](https://www.jianshu.com/p/87f101d8ec41)
- [Spring Boot 启动过程分析](https://www.jianshu.com/p/dc12081b3598)