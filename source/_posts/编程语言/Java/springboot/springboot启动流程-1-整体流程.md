---
title: springboot启动流程-01-启动流程
date: 2021-04-16 09:30:13
tags: [java,springboot,startup]
---

由于该系统是底层系统，以微服务形式对外暴露dubbo服务，所以本流程中SpringBoot不基于jetty或者tomcat等容器启动方式发布服务，而是以执行程序方式启动来发布

# SpringBoot 启动过程

![img](https://upload-images.jianshu.io/upload_images/6912735-51aa162747fcdc3d.png)

启动流程主要分为三个部分，

1. 第一部分进行 `SpringApplication` 的初始化模块
   1. 配置一些基本的**环境变量、资源、构造器、监听器**。
2. 第二部分实现了应用具体的启动方案，包括启动流程的监听模块、加载配置环境模块、及核心的创建上下文环境模块

3. 第三部分是自动化配置模块，该模块作为springboot自动配置核心，在后面的分析中会详细讨论。在下面的启动程序中我们会串联起结构中的主要功能。





## @SpringApplication

 每个SpringBoot程序都有一个主入口，也就是main方法，main里面调用SpringApplication.run()启动整个spring-boot程序。SpringBoot要求该main方法所在类必须使用@SpringBootApplication注解，以及@ImportResource注解(if need)

@SpringBootApplication包括三个注解：

| 注解                     | 功能                                                         |
| ------------------------ | ------------------------------------------------------------ |
| @EnableAutoConfiguration | 自动配置：从 `classpath` 中搜寻所有的 `META-INF/spring.factories` 配置文件，并将其中 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 对应的配置项通过**反射实例化**为对应的标注了 `@Configuration` 的 `JavaConfig` 形式的 IoC 容器配置类，然后汇总为一个并加载到 IoC 容器。 |
| @SpringBootConfiguration | 源码内部其实是`@Configuration`：被标注的类等于在spring的XML配置文件中`applicationContext.xml`，装配所有bean事务，提供了一个spring的上下文环境Context |
| @ComponentScan           | 组件扫描，可自动发现和装配Bean，默认扫描SpringApplication的run(xxx.class)中, xxx.class所在的包路径下文件，所以最好将该启动类放到根包路径下。`@ComponentScan`通常与`@Configuration`一起配合使用，相当于xml里面的`<context:component-scan>`，用来告诉Spring需要扫描哪些包或类。如果不设值的话默认扫描@ComponentScan注解所在类的同级类和同级目录下的所有类  , *所以对于一个Spring Boot项目，一般会把入口类放在顶层目录中，这样就能够保证源码目录下的所有类都能够被扫描到*。 |



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
  
  //key-1: 获取SpringApplicationRunListeners
  SpringApplicationRunListeners listeners = this.getRunListeners(args);
  
  // 通知监听者，开始启动
  listeners.starting();

  Collection exceptionReporters;
  try {
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    
     // KEY 2 - 根据SpringApplicationRunListeners以及参数来准备环境env
    ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
    
    this.configureIgnoreBeanInfo(environment);
    Banner printedBanner = this.printBanner(environment);
    
    //key-3:  创建`应用配置上下文ConfigurableApplicationContext(即run方法的返回对象)
    context = this.createApplicationContext();
    exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
    
    // KEY 4 - Spring上下文前置处理
    this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    // KEY 5 - Spring上下文刷新, 实现spring-boot-starter-(mybatis、redis等)自动化配置的关键，包括spring.factories的加载，bean的实例化等核心工作
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













## 参考

- [SpringBoot启动流程解析](https://www.jianshu.com/p/87f101d8ec41)
- [Spring Boot 启动过程分析](https://www.jianshu.com/p/dc12081b3598)