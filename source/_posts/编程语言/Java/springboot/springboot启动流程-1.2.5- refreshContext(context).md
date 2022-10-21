---
title: springboot启动流程-01-启动流程
date: 2021-04-16 09:30:13
tags: [java,springboot,startup]
---

https://segmentfault.com/a/1190000020494692

### SpringApplication#refreshContext

refreshContext方法刷新应用上下文并进行自动化配置模块加载，也就是上文提到的SpringFactoriesLoader根据指定classpath加载META-INF/spring.factories文件的配置，实现自动配置核心功能。

```java
# SpringApplication.class
private void refreshContext(ConfigurableApplicationContext context) {
  // 由于这里需要调用父类一系列的refresh操作，涉及到了很多核心操作
    refresh(context);

    // 注册一个关闭容器时的钩子函数
    if (this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}

// 调用父类的refresh方法完成容器刷新的基础操作
protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    ((AbstractApplicationContext)applicationContext).refresh();
}
```



### AbstractApplicationContext#refresh

```java
# AbstractApplicationContext.class
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    //第一步：容器刷新前的准备，设置上下文状态，获取属性，验证必要的属性等
    prepareRefresh();

    //第二步：获取新的beanFactory，销毁原有beanFactory、为每个bean配置信息生成BeanDefinition等,注意此处是获取新的，销毁旧的，这就是刷新的意义
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    //第三步：配置标准的beanFactory，设置ClassLoader，设置SpEL表达式解析器等
    prepareBeanFactory(beanFactory);
    try {
      //第四步：在所有的beanDenifition加载完成之后，bean实例化之前执行。比如在beanfactory加载完成所有的bean后，想修改其中某个bean的定义，或者对beanFactory做一些其他的配置，就可以在子类中对beanFactory进行后置处理。
      postProcessBeanFactory(beanFactory);

      //第五步：实例化并调用所有注册的beanFactory后置处理器（实现接口BeanFactoryPostProcessor的bean） 
      invokeBeanFactoryPostProcessors(beanFactory);

      //第六步：实例化和注册beanFactory中扩展了BeanPostProcessor的bean。
      //例如： 
      //AutowiredAnnotationBeanPostProcessor(处理被@Autowired注解修饰的bean并注入)
      //RequiredAnnotationBeanPostProcessor(处理被@Required注解修饰的方法)
      //CommonAnnotationBeanPostProcessor(处理@PreDestroy、@PostConstruct、@Resource等多个注解的作用)等。
      registerBeanPostProcessors(beanFactory);

      //第七步：初始化国际化工具类MessageSource
      initMessageSource();

      //第八步：初始化应用事件广播器。这是观察者模式的典型应用。我们知道观察者模式由主题Subject和Observer组成。广播器相当于主题Subject，其包含多个监听器。当主题发生变化时会通知所有的监听器。初始化应用消息广播器，并放入"ApplicationEventMulticaster" Bean中
      initApplicationEventMulticaster();

      //第九步：这个方法在AnnotationApplicationContex上下文中没有实现，留给子类来初始化其他的Bean，是个模板方法，在容器刷新的时候可以自定义逻辑（子类自己去实现逻辑），不同的Spring容器做不同的事情
      onRefresh();

      //第十步：注册监听器，并且广播early application events,也就是早期的事件
      registerListeners();

      //第十一步：初始化剩下的单例（非懒加载的单例类）（并invoke BeanPostProcessors）
      //实例化所有剩余的（非懒加载）单例Bean。（也就是我们自己定义的那些Bean）
      //比如invokeBeanFactoryPostProcessors方法中根据各种注解解析出来的类，在这个时候都会被初始化，扫描的@Bean之类的，
      //实例化的过程各种BeanPostProcessor开始起作用
      finishBeanFactoryInitialization(beanFactory);

      //第十二步：完成刷新过程，通知生命周期处理器lifecycleProcessor完成刷新过程，同时发出ContextRefreshEvent通知别人
      //refresh做完之后需要做的其他事情
      //清除上下文资源缓存（如扫描中的ASM元数据）
      //初始化上下文的生命周期处理器，并刷新（找出Spring容器中实现了Lifecycle接口的bean并执行start()方法）。
      //发布ContextRefreshedEvent事件告知对应的ApplicationListener进行响应的操作
      finishRefresh();
    }

    catch (BeansException ex) {
      if (logger.isWarnEnabled()) {
        logger.warn("Exception encountered during context initialization - " +
                    "cancelling refresh attempt: " + ex);
      }
      //如果刷新失败那么就会将已经创建好的单例Bean销毁掉
      destroyBeans();

      //重置context的活动状态 告知是失败的
      cancelRefresh(ex);

      //抛出异常
      throw ex;
    }

    finally {
      // 失败与否，都会重置Spring内核的缓存。因为可能不再需要metadata给单例Bean了。
      resetCommonCaches();
    }
  }
}
```



### AbstractApplicationContext#prepareRefresh

```java
protected void prepareRefresh() {
        //记录容器启动时间，然后设立对应的标志位
        this.startupDate = System.currentTimeMillis();
        this.closed.set(false);
        this.active.set(true);
        // 打印info日志：开始刷新当前容器了
        if (logger.isInfoEnabled()) {
            logger.info("Refreshing " + this);
        }

        // 这是扩展方法，由子类去实现，可以在验证之前为系统属性设置一些值可以在子类中实现此方法
        // 因为我们这边是AnnotationConfigApplicationContext，可以看到不管父类还是自己，都什么都没做，所以此处先忽略
        initPropertySources();

        //属性文件验证，确保需要的文件都已经放入环境中
        getEnvironment().validateRequiredProperties();

        //初始化容器，用于装载早期的一些事件
        this.earlyApplicationEvents = new LinkedHashSet<>();
    } 
```

### AbstractApplicationContext#obtainFreshBeanFactory
