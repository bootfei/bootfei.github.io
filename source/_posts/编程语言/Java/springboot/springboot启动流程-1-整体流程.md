---
title: springboot启动流程-01-启动流程
date: 2021-04-16 09:30:13
tags: [java,springboot,startup]
---

由于该系统是底层系统，以微服务形式对外暴露dubbo服务，所以本流程中SpringBoot不基于jetty或者tomcat等容器启动方式发布服务，而是以执行程序方式启动来发布

# SpringBoot 启动过程

<img src="/Users/qifei/Documents/blog/source/_posts/编程语言/Java/springboot/springboot启动流程图.png" alt="img" style="zoom:350%;" />

启动流程主要分为三个部分，

1. 第一部分进行 `SpringApplication` 的初始化模块: 配置一些基本的**环境变量、资源、构造器、监听器**。
2. 第二部分实现了应用具体的启动方案，包括启动流程的监听模块、加载配置环境模块、及核心的创建上下文环境模块
3. 第三部分是自动化配置模块，该模块作为springboot自动配置核心





## @SpringApplication

 每个SpringBoot程序都有一个主入口，也就是main方法，main里面调用SpringApplication.run()启动整个spring-boot程序。SpringBoot要求该main方法所在类必须使用@SpringBootApplication注解，以及@ImportResource注解(if need)

![](https://img2018.cnblogs.com/blog/1158841/201907/1158841-20190709114128801-171612088.png)



## SpringApplication.class源码

```java
// SpringApplication.class

// 构造函数：在run()方法链路：2nd step中被调用
public SpringApplication(Object... sources) {
    initialize(sources);
}

// initailize()方法 在run()方法链路：2nd step 构造函数中被调用
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

//----------------分割线-----------------

//run()方法链路：1st step
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
  return run(new Class[]{primarySource}, args);
}

//run()方法链路：2nd step
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
  //我们可以发现其构造方法在这，其实调用了一个初始化的initialize方法
  return (new SpringApplication(primarySources)).run(args);
}

//run()方法链路：3nd step
public ConfigurableApplicationContext run(String... args) {
  // 此类通常用于监控开发过程中的性能，而不是生产应用程序的一部分。
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();
  
 
  ConfigurableApplicationContext context = null;
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
  
   // 设置java.awt.headless系统属性，默认为true 
    // Headless模式是系统的一种配置模式。在该模式下，系统缺少了显示设备、键盘或鼠标。
  this.configureHeadlessProperty();
  
  //key-1: 获取SpringApplicationRunListeners，并通知监听者，开始启动
  SpringApplicationRunListeners listeners = this.getRunListeners(args);
  listeners.starting();

  Collection exceptionReporters;
  try {
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    
    // key-2: 根据SpringApplicationRunListeners以及参数来准备环境env
    ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
    
    this.configureIgnoreBeanInfo(environment);
    
    //（忽略）banner小彩蛋
    Banner printedBanner = this.printBanner(environment);
    
    //key-3:  创建`应用配置上下文ConfigurableApplicationContext(即run方法的返回对象)
    context = this.createApplicationContext();
    exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
    
    //key-4: Spring上下文前置处理
    this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    //key-5: Spring上下文刷新, 实现spring-boot-starter-(mybatis、redis等)自动化配置的关键，包括spring.factories的加载，bean的实例化等核心工作
    this.refreshContext(context);
    
    // KEY 6 - Spring上下文后置处理
    this.afterRefresh(context, applicationArguments);
    
    // 发出结束执行的事件
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



private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
  ConfigurableEnvironment environment = this.getOrCreateEnvironment();
  this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
  //3. 配置`变量environment对象`加入到监听器对象中`SpringApplicationRunListeners`
  listeners.environmentPrepared((ConfigurableEnvironment)environment);
  this.bindToSpringApplication((ConfigurableEnvironment)environment);
  if (this.webApplicationType == WebApplicationType.NONE) {
    environment = (new EnvironmentConverter(this.getClassLoader())).convertToStandardEnvironmentIfNecessary((ConfigurableEnvironment)environment);
  }

  ConfigurationPropertySources.attach((Environment)environment);
  return (ConfigurableEnvironment)environment;
}
```

## 1st step:构造函数使用initialize()实例化

构造函数中调用了initialize()方法

```java
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
```

## 2nd step: run()返回context

key-1: 获取`SpringApplicationRunListeners`

key 2: 加载SpringBoot配置环境`ConfigurableEnvironment`

key 3: 创建应用配置上下文 `ConfigurableApplicationContext` (即run方法的返回对象)

key 4:  `ConfigurableApplicationContext`前置处理

key 5:  `ConfigurableApplicationContext`刷新，实现spring-boot-starter-(mybatis、redis等)自动化配置的关键，包括spring.factories的加载，bean的实例化等核心工作。

key 6:  `ConfigurableApplicationContext`后置处理
