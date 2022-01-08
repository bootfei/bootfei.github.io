---
title: springboot启动流程-01-启动流程
date: 2021-04-16 09:30:13
tags: [java,springboot,startup]
---



## SpringApplication.class源码

```java
// SpringApplication.class


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
    
     // KEY-2: 根据SpringApplicationRunListeners以及参数来准备环境env
    ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
    
    this.configureIgnoreBeanInfo(environment);
    Banner printedBanner = this.printBanner(environment);
    
    //key-3:  创建`应用配置上下文ConfigurableApplicationContext(即run方法的返回对象)
    context = this.createApplicationContext();
    
    exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
    
    // KEY 4 - Spring上下文前置处理, prepareContext方法将listeners、environment、applicationArguments、banner等重要组件与上下文对象关联
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



```



### Key1: 获取listeners

### Key2: prepareEnvironment()准备env

```java
//SpringApplicaiton.class

//key-2详情
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
  ConfigurableEnvironment environment = this.getOrCreateEnvironment();
  this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
  // 配置`变量environment对象`加入到监听器对象中`SpringApplicationRunListeners`
  listeners.environmentPrepared((ConfigurableEnvironment)environment);
  this.bindToSpringApplication((ConfigurableEnvironment)environment);
  
  if (this.webApplicationType == WebApplicationType.NONE) {
    environment = (new EnvironmentConverter(this.getClassLoader())).convertToStandardEnvironmentIfNecessary((ConfigurableEnvironment)environment);
  }

  ConfigurationPropertySources.attach((Environment)environment);
  return (ConfigurableEnvironment)environment;
}
```

### Key3: 创建ConfigurableApplicationContext

```java
    //SpringApplicaiton.class
protected ConfigurableApplicationContext createApplicationContext() {
  //首先获取显式设置的应用上下文applicationContextClass
  Class<?> contextClass = this.applicationContextClass;
  
  //如果不存在，再加载默认的环境配置（通过是否是web environment判断）
  if (contextClass == null) {
    try {
      switch(this.webApplicationType) {
        case SERVLET:
          contextClass = Class.forName("org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext");
          break;
        case REACTIVE:
          contextClass = Class.forName("org.springframework.boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext");
          break;
        default:
          contextClass = Class.forName("org.springframework.context.annotation.AnnotationConfigApplicationContext");
      }
    } catch (ClassNotFoundException var3) {
      throw new IllegalStateException("Unable create a default ApplicationContext, please specify an ApplicationContextClass", var3);
    }
  }

  //最后通过BeanUtils实例化上下文对象
  return (ConfigurableApplicationContext)BeanUtils.instantiateClass(contextClass);
}
```

1. 方法会先获取显式设置的应用上下文(applicationContextClass)，
2. 如果不存在，再加载默认的环境配置（通过是否是web environment判断），默认选择AnnotationConfigApplicationContext注解上下文（通过扫描所有注解类来加载bean），
3. 最后通过BeanUtils实例化context并返回context

ConfigurableApplicationContext类图如下：

![img](https://upload-images.jianshu.io/upload_images/6912735-797f3d2c57b625bc.png)

主要看其继承的两个方向：

LifeCycle：生命周期类，定义了start启动、stop结束、isRunning是否运行中等生命周期空值方法

ApplicationContext：应用上下文类，其主要继承了beanFactory(bean的工厂类)



### Key4:prepareContext()将listeners、environment、applicationArguments、banner等重要组件与上下文对象关联



### Key5: refresh() Spring上下文刷新, 实现spring-boot-starter-(mybatis、redis等)自动化配置的关键，包括spring.factories的加载，bean的实例化等核心工作