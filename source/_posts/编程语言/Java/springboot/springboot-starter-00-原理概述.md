---
title: springboot-定时任务框架
date: 2021-04-12 19:22:20
tags: [springboot, spring starter]
---



#### @EnableAutoConfiguration

@Import(AutoConfigurationImportSelector.class)



AutoConfigurationImportSelector是在哪里被Spring IOC解析的呢？



AutoConfigurationImportSelector#getCandiate()

```java
   protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
```

SpringBootFactoryLoader#loadFactoryNames

```java
public abstract class SpringFactoriesLoader {
    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
    private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);
    private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap();
...
}
```

META-INF/spring.factories在

![springboot-starter-SpringBootFactoryLoader使用的META-INFO:Spring.factories](/Users/qifei/Documents/blog/source/_posts/编程语言/Java/springboot/springboot-starter-SpringBootFactoryLoader使用的META-INFO:Spring.factories.png)

```

```





#### 解析enableAutoConfiguration.class的地址

SB.class#run()

```java
ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);


listeners.environmentPrepared((ConfigurableEnvironment)environment);
```

SpringApplicationRunListeners#environmentPrepared

```
    public void environmentPrepared(ConfigurableEnvironment environment) {
        Iterator var2 = this.listeners.iterator();

        while(var2.hasNext()) {
            SpringApplicationRunListener listener = (SpringApplicationRunListener)var2.next();
            listener.environmentPrepared(environment);
        }

    }
```

EventPublishingRunListenerz#environmentPrepared

```
    
    public void environmentPrepared(ConfigurableEnvironment environment) {
        this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
    }
```

最后看到，SimpleApplicationEventMulticaster#doInvokeListener中

-> listen.onApplicaitonEvent(event)，ConfigFileApplicaitonListener.class也是个listener，并且监听了

![image-20211201212957091](/Users/qifei/Library/Application Support/typora-user-images/image-20211201212957091.png)







ConfigFileApplicaitonListener.class同时也是个postProcessor

![image-20211201213539624](/Users/qifei/Library/Application Support/typora-user-images/image-20211201213539624.png)

ConfigFileApplicaitonListener.class会加载所有的配置文件，包括yml中include的文件

![image-20211202085003762](/Users/qifei/Library/Application Support/typora-user-images/image-20211202085003762.png)
