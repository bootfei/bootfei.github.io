---
title: springboot启动流程-01-启动流程
date: 2021-04-16 09:30:13
tags: [java,springboot,startup]
---



# 参考阅读

https://mp.weixin.qq.com/s/bAWhkNnsD0S0Hv7E-AxjIA

# SpringBoot 启动过程



启动流程主要分为三个部分，

1. 第一部分进行 `SpringApplication` 的初始化模块: 配置一些基本的**环境变量、资源、构造器、监听器**。
2. 第二部分实现了应用具体的启动方案，包括启动流程的监听模块、加载配置环境模块、及核心的创建上下文环境模块
3. 第三部分是自动化配置模块，该模块作为springboot自动配置核心





## @SpringApplication

 每个SpringBoot程序都有一个主入口，也就是main方法，main里面调用SpringApplication.run()启动整个spring-boot程序。SpringBoot要求该main方法所在类必须使用@SpringBootApplication注解，以及@ImportResource注解(if need)





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





# 链路1：Bean的创建链路

`AbstractBeanFactory`#refresh()

-> `finishBeanFactoryInitialization`#finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) 方法

->  `beanFactory`.preInstantiateSingletons()，在这里对所有的 beanDefinitionNames 一一遍历，进行 bean 实例化和组装：

```java
@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof SmartFactoryBean<?> smartFactoryBean && smartFactoryBean.isEagerInit()) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton smartSingleton) {
				StartupStep smartInitialize = getApplicationStartup().start("spring.beans.smart-initialize")
						.tag("beanName", beanName);
				smartSingleton.afterSingletonsInstantiated();
				smartInitialize.end();
			}
		}
	}
```

​	这个 beanDefinitionNames 列表的顺序就决定了 Bean 的创建顺序，那么这个 beanDefinitionNames 列表又是怎么来的？答案是 ConfigurationClassPostProcessor 通过扫描你的代码和注解生成的，将 Bean 扫描解析成 Bean 定义（BeanDefinition），同时将 Bean 定义（BeanDefinition）注册到 BeanDefinitionRegistry 中，才有了 beanDefinitionNames 列表。

->`AbstractBeanFactory`#getBean()

```java
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
```

->`AbstractBeanFactory`#doGetBean()

现在通过源码分析，深入理解下Spring如何运用三级缓存解决循环依赖。Spring创建Bean的核心代码doGetBean中，在实例化bean之前，会先尝试从三级缓存获取bean，这也是Spring解决循环依赖的开始。

我们假设现在有这样的场景AService依赖BService，BService依赖AService

一开始加载AService Bean首先依次从一二三级缓存中查找是否存在beanName=AService的对象。

```java

// AbstractBeanFactory.java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    final String beanName = transformedBeanName(name);
    // 1.尝试从缓存中获取bean，AService还没创建三级缓存都没命中
    Object sharedInstance = getSingleton(beanName);
    if (mbd.isSingleton()) {

        sharedInstance = getSingleton(beanName,    () -> {  //注意此处参数是一个lambda表达式即参数传入的是ObjectFactory类型一个匿名内部类对象
                                                    try {
                                                        return createBean(beanName, mbd, args);  // 
                                                    }
                                                    catch (BeansException ex) {}
                                                });
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
}
```



-> `AbstractAutowireCapableBeanFactory`类的`doCreateBean`方法是创建`bean`的开始，

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    // Bean初始化第一步：默认调用无参构造实例化Bean
    // 如果是只有带参数的构造方法，构造方法里的参数依赖注入，就是发生在这一步
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // bean创建第二步：填充属性（DI依赖注入发生在此步骤）
        populateBean(beanName, mbd, instanceWrapper);
        // bean创建第三步：调用初始化方法，完成bean的初始化操作（AOP的第三个入口）
        // AOP是通过自动代理创建器AbstractAutoProxyCreator的postProcessAfterInitialization()
//方法的执行进行代理对象的创建的,AbstractAutoProxyCreator是BeanPostProcessor接口的实现
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        // ...
    }
    // ...
```



## 案例:三层缓存

https://mp.weixin.qq.com/s/dSRQBSG42MYNa992PvtnJA



# 链路2：BeanFactory扫描beanDefinition的链路

<img src="https://pic3.zhimg.com/80/v2-76723f4b445923f8a4296ba329051d1e_1440w.webp" style="zoom:75%;" />

# 链路3：BeanFactory创建Bean的链路
