---
title: springboot启动流程-01-启动流程
date: 2021-04-16 09:30:13
tags: [java,springboot,startup]
---

obtainFreshBeanFactory()方法会解析所有Spring配置文件（通常我们会放在 resources 目录下），将所有 Spring 配置文件中的 bean 定义封装成 BeanDefinition，加载到 BeanFactory中。



> 比如扫描xxx.xml文件，如果解析到<context:component-scan base-package="" /> 注解时，会扫描 base-package 指定的目录，将该目录下使用指定注解（@Controller、@Service、@Component、@Repository）的 bean 定义也同样封装成 BeanDefinition，加载到 BeanFactory 中。



## AbstractApplicationContext#obtainFreshBeanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        //初始化BeanFactory
        this.refreshBeanFactory();
        //返回初始化之后的BeanFactory
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (logger.isDebugEnabled()) {
            logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
        }
        return beanFactory;
    }


		protected final void refreshBeanFactory() throws BeansException {
        //判断是否已经存在BeanFactory，存在则销毁所有Beans，并且关闭BeanFactory
        if (hasBeanFactory()) {
            //销毁所有的bean
            destroyBeans();
            //关闭并销毁BeanFactory
            closeBeanFactory();
        }
        try {
           //创建具体的beanFactory，这里创建的是DefaultListableBeanFactory，最重要的beanFactory spring注册及加载bean就靠它
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());
            //定制BeanFactory，包括是否允许覆盖同名称的不同定义的对象以及循环依赖
            customizeBeanFactory(beanFactory);
            //这个就是最重要的了，加载所有的Bean配置信息（属于模版方法，由子类去实现加载的方式）
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        }
        catch (IOException ex) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
        }
    }



		protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
        //allowBeanDefinitionOverriding属性是指是否允对一个名字相同但definition不同进行重新注册，默认是true。
        if (this.allowBeanDefinitionOverriding != null) {
            beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
         //allowCircularReferences属性是指是否允许Bean之间循环引用，默认是true.
        if (this.allowCircularReferences != null) {
            beanFactory.setAllowCircularReferences(this.allowCircularReferences);
        }
    }

    protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;

```



### AbstractApplicationContext#refreshBeanFactory

![image-20211125190407914](/Users/qifei/Library/Application Support/typora-user-images/image-20211125190407914.png)

由于在我们当前容器`AnnotationConfigServletWebServerApplicationContext`的所有父类（或者父类的父类）中，只有`GenericApplicationContext`重写了这个方法，所以实际调用的是`GenericApplicationContext`的`refreshBeanFactory`。

`refreshBeanFactory`方法中其实就做了两件事，一个是设置`refreshed`的值，这里用到了`java`的`CAS`机制，是线程安全的一个机制，如果实际值与预期值（`expect`）一致，则将值设置为更新值（`update`），这一块属于多线程线程安全领域的内容；另一个操作就是设置`beanFactory`的`serializationId`，这个值默认设置的是`spring.application.name`的值。

```java
#GenericApplicationContext
    protected final void refreshBeanFactory() throws IllegalStateException {
        if (!this.refreshed.compareAndSet(false, true)) {
            throw new IllegalStateException("GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
        } else {
            this.beanFactory.setSerializationId(this.getId());
        }
    }
```



### 实现loadBeanDefinitions()的子类有多种，

- AbstractXmlApplicationContext类提供了基于XML的加载实现
- AnnotationConfigWebApplicationContext类提供了在webapp的场景下基于注解配置的加载实现，
- XmlWebApplicationContext类提供了在webapp场景下基于xml配置的加载实现，
- XmlPortletApplicationContext提供了在portalet下基于xml配置的加载实现，
- GroovyWebApplicationContext类提供了基于groovy脚本配置的加载实现。
- ![clipboard.png](https://segmentfault.com/img/bVbx9Iv)

下面以AnnotationConfigWebApplicationContext#loadBeanDefinitions()方法为例看下代码。

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {
        //初始化这个脚手架 其实就是直接new出实例
       AnnotatedBeanDefinitionReader reader = getAnnotatedBeanDefinitionReader(beanFactory);
        ClassPathBeanDefinitionScanner scanner = getClassPathBeanDefinitionScanner(beanFactory);
        // 生成Bean的名称的生成器，如果自己没有setBeanNameGenerator（可以自定义）,这里目前为null
        BeanNameGenerator beanNameGenerator = getBeanNameGenerator();
        if (beanNameGenerator != null) {
            reader.setBeanNameGenerator(beanNameGenerator);
            scanner.setBeanNameGenerator(beanNameGenerator);
            //若我们注册了beanName生成器，那么就会注册进容器里面
            	beanFactory.registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
        }
        
        //这是给reader和scanner注册scope的解析器  此处为null
        ScopeMetadataResolver scopeMetadataResolver = getScopeMetadataResolver();
        if (scopeMetadataResolver != null) {
            reader.setScopeMetadataResolver(scopeMetadataResolver);
            scanner.setScopeMetadataResolver(scopeMetadataResolver);
        }

        //我们可以自己指定annotatedClasses 配置文件,同时也可以交给下面扫描
        if (!this.annotatedClasses.isEmpty()) {
            // 这里会把所有的配置文件输出=======info日志  请注意观察控制台
            if (logger.isDebugEnabled()) {
                logger.debug("Registering annotated classes: [" +
                        StringUtils.collectionToCommaDelimitedString(this.annotatedClasses) + "]");
            }
            // 若是指明的Bean，就交给reader去处理
            reader.register(ClassUtils.toClassArray(this.annotatedClasses));
        }

        // 也可以是包扫描的方式，扫描配置文件的Bean
        if (!this.basePackages.isEmpty()) {
            if (logger.isDebugEnabled()) {
                logger.debug("Scanning base packages: [" +
                        StringUtils.collectionToCommaDelimitedString(this.basePackages) + "]");
            }
            scanner.scan(StringUtils.toStringArray(this.basePackages));
        }

        // 此处的意思是，也可以以全类名的形式注册。比如可以调用setConfigLocations设置（这在xml配置中使用较多）  可以是全类名，也可以是包路径
        String[] configLocations = getConfigLocations();
        if (configLocations != null) {
            for (String configLocation : configLocations) {
                try {
                    Class<?> clazz = ClassUtils.forName(configLocation, getClassLoader());
                    if (logger.isTraceEnabled()) {
                        logger.trace("Registering [" + configLocation + "]");
                    }
                    reader.register(clazz);
                }
                catch (ClassNotFoundException ex) {
                    if (logger.isTraceEnabled()) {
                        logger.trace("Could not load class for config location [" + configLocation +
                                "] - trying package scan. " + ex);
                    }
                    int count = scanner.scan(configLocation);
                   // 发现不是全类名，那就当作包扫描
                    if (count == 0 && logger.isDebugEnabled()) {
                        logger.debug("No annotated classes found for specified class/package [" + configLocation + "]");
                    }
                }
            }
        }
    }
```

该方法主要是解析我们项目配置的 application.xml、xxx.xml 定义的import、bean、resource、profile或扫描注解 将其属性封装到BeanDefinition 对象中。

上面提到的 “加载到 BeanFactory 中” 的内容主要指的是添加到以下3个缓存：

- beanDefinitionNames缓存：所有被加载到 BeanFactory 中的 bean 的 beanName 集合。

- beanDefinitionMap缓存：所有被加载到 BeanFactory 中的 bean 的 beanName 和 BeanDefinition 映射 Map<String, BeanDefinition> (beanName, beanDefinition）
- aliasMap缓存：所有被加载到 BeanFactory 中的 bean 的 beanName 和别名映射。

现在BeanFactory已经创建完成了，并且Config配置文件的Bean定义已经注册完成了（备注：其它单例Bean是还没有解析的），下面的步骤大都把BeanFactory传进去了，都是基于此Bean工厂的操作。
