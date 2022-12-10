---
title: spring-02-ioc和di
date: 2020-10-17 19:52:16
categories: [java,spring]
tags:
---



## 容器实例的创建: IOC

Spring Context容器的入口是`org.springframework.context.support.AbstractApplicationContext#refresh`，这里是整个IoC的完整过程。

- 构建`BeanFactory`
- 注册后置处理器`BeanPostProcessor`。**Spring framework**中注册6个**BeanPostProcessor**的过程
- 国际化，感兴趣的事件
- 创建Bean，根据`BeanPostProcessor`完成注入

### 高级容器ApplicationContext

各种启动方式

```java
//基于xml的配置
ClassPathXmlApplicationContext context=new ClassPathXmlApplicationContext(“classpath:spring.xml”);
//基于java的配置
AnnotaitionConfigApplicationContext context=new AnnotationConfigApplicationContext(“com.star.config.KnightConfig.class”); 

//SpringBoot启动方式
SpringApplicationContext.run(xxx.class, args);
```

[从上图中可以看出 ApplicationContext 继承了 BeanFactory，这也说明了 Spring 容器中运行的主体对象是 Bean]()

#### ApplicationContext启动流程

- SpringBoot启动方式

SpringBoot中SpringApplication.run()是如何识别用哪种配置方式 or 哪种ApplicationContext呢？

```java
public ConfigurableApplicationContext run(String... args){
	...
    context = createApplicationContext(); //对ApplicationContext进行选择
  	refreshContext(context); //这里面其实就是ApplicationContext#refresh
  ...
}
```

```java
   protected ConfigurableApplicationContext createApplicationContext() {
      ...
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
            
			...
        return (ConfigurableApplicationContext)BeanUtils.instantiateClass(contextClass);
    }
```



- xml方式

应用上下文准备就绪之后，我们就可以调用BeanFactory的getBean("xxx.class")方法从Spring容器中获取bean。



#### AbstractApplicationContext#refresh()

> 注意：不管spring的xml或者java启动方式、还是spring boot和cloud，最终都会调 [AbstractApplicationContext的refresh方法]() ，而这个方法才是我们真正的入口。
>

```java
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
            // STEP 2： 非常重要！！！
            // a） 创建基础IoC容器（DefaultListableBeanFactory） 
            // b） 加载解析XML文件（最终存储到Document对象中）
            // c） 读取Document对象，并完成BeanDefinition的加载和注册工作
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);
				beanPostProcess.end();

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
                // STEP 11： 实例化剩余的单例bean（非懒加载方式） 非常重要！！！
                // 注意事项：Bean的IoC、DI和AOP都是发生在此步骤
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
				contextRefresh.end();
			}
		}
	}
```

12个步骤step：step2和step11是基础容器的内容，其他step都是高级容器的特有功能

这段代码主要包含这样几个步骤：

- Step2: 构建 BeanFactory
- 注册可能感兴趣的事件。
- 创建 Bean 实例对象。
- 触发被监听的事件。

<!--这一步非常重要，就是finishBeanFactoryInitialization(beanFactory)，高级容器Context与基础容器BeanFactory建立了联系-->

#### 子类1：AnnotationConfigApplicationContext

> ApplicationContext上下文，接受一个Component类做为输入参数。例如`@Configuration`注解修饰的类。即支持基于Java的配置类。
>
> `AnnotationConfigApplicationContext`间接实现了`ApplicationContext`接口

- 持有AnnotatedBeanDefinitionReader reader和ClassPathBeanDefinitionScanner scanner两个引用
- 4种构造函数

```java
/**
 *  Spring Boot采用该方式实例化Application Context(通过反射调用无参构造方法)。
 */
public AnnotationConfigApplicationContext() {
    // 读取器，用于支持以编程的方式注册Bean类,配置各种PostProcessor
    this.reader = new AnnotatedBeanDefinitionReader(this);
    // BeanDefinition扫描器,检测指定path下的可能的Bean，并将其注册到BeanFactory或者ApplicationContext中.可以用来扫描包和类,将其转换为BeanDefinition
    // 通过AnnotationConfigApplicationContext(Class<?>... componentClasses)方式启动时，其不是真正的scanner,真正的scanner是org.springframework.context.annotation.ComponentScanAnnotationParser.parse()方法创建的scanner
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}

public AnnotationConfigApplicationContext(DefaultListableBeanFactory beanFactory) {
    //父类GenericApplicationContext无参数构造方法默认会创建DefaultListableBeanFactory
    super(beanFactory);
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}

/**
 * 创建AnnotationConfigApplicationContext，生成新的bean definitions并从给定的组件类且自动刷新context
 */
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();
    register(componentClasses); //reader将启动类注册到BeanFactory中。
    refresh();
}

// 传递basePackages，例如 basePackages=com.abc
public AnnotationConfigApplicationContext(String... basePackages) {
    this();
    // 扫描指定的包路径，classpath*:com/abc/**/*.class下的所有class类,将符合条件的类生成BeanDefinition,
    // 遍历并注册到BeanFactory中
    scan(basePackages);
    // AbstractApplicationContext.refresh()方法，是Spring启动的核心
    refresh();
}
```



#### 子类2：AnnotationConfigServletWebServerAC



#### 子类3：AnnotationConfigReactiveWebServerAC



#### AnnotatedBeanDefinitionReader

> 是一个适配器，用于支持以编程的方式注册Bean类，其通过编程的方式其支持以编程的方式注册Bean类，会添加一些注解支持。例如
>
> - `ConfigurationClassPostProcessor`支持`@Configuration`
> - `AutowiredAnnotationBeanPostProcessor`支持`@Autowired`、`@Value`
> - `CommonAnnotationBeanPostProcessor`支持`@PostConstruct`、`@PreDestory`、`@Resource`等
> - `EventListenerMethodProcessor`支持`@EventListener`

- 在AnnotationConfigApplicationContext的构造函数中被调用，register(...)就是reader.register(...)

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();
    register(componentClasses); //reader将启动类注册到BeanFactory中。
    refresh();
}
```

- AnnotationConfigUtils.registerAnnotationConfigProcessors()方法，在构造方法中被调用；添加各种PostProcessor

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment){
	...
	AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```



```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
    BeanDefinitionRegistry registry, @Nullable Object source) {

    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
    ...
    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

    // 添加@Configuration注解后置处理器:ConfigurationClassPostProcessor(BeanFactoryPostProcessor)
    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        def.setSource(source);
        // 将ConfigurationClassPostProcessor以CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME做为BeanName
        // 放入到beanDefinitionMap中，并返回一个holder
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 添加@Autowired、@Value注解后置处理器:AutowiredAnnotationBeanPostProcessor
    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        ...
    }
    
    ...
}
```

- AnnotatedBeanDefinitionReader.doRegisterBean()：

> - 在reader.register(...)方法中被循环调用，注册该`componentClasses`, 启动类其实就是个`componentClasses`,因为被@Component修饰
> - 将BeanDefinition放入到Registry抽象类的Map<String,BeanDefinition> beanDefinitionMap

```java
// 将启动类作为一个Bean加入到beanDefinitionMap中去
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
                                @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
                                @Nullable BeanDefinitionCustomizer[] customizers) {
    // 根据提供的类创建一个BeanDefinition
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    // 判断该类是否跳过解析，主要是@Condition相关
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setInstanceSupplier(supplier);
    // 作用域scope
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    // beanName，根据BeanNameGenerator
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
    // 处理类的通用注解,@Primary、@Lazy、@DependsOn、@Role、@Description
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    // @Qualifier注解相关,如果向容器中注册Bean时，当使用类@Qualifier注解时
    if (qualifiers != null) {
       ....
    }
  	// 自定义注解相关
    if (customizers != null) {
       ....
    }

    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 将beanDefinition放入到this.registry中的Map<String,BeanDefinition> beanDefinitionMap中！！！
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```



#### ClassPathBeanDefinitionScanner

> `ClassPathBeanDefinitionScanner`的作用是读取class后缀文件，然后包装成BeanDefinition，注册时BeanFactory中。

在AnnotationConfigApplicationContext的构造函数中被调用，scanner(...)就是scanner.scan(...)

```java
public AnnotationConfigApplicationContext(String... basePackages) {
    this();
    scan(basePackages);
    refresh();
}
```

- scanner.scan()


```java
// 在指定的packages中执行扫描
public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
	// 调用的是其父类 ClassPathScanningCandidateComponentProvider
    doScan(basePackages);

    // Register annotation config processors, if necessary.
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }

    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
```

- doScan()方法

```java
//扫描指定packages中的类，返回符合条件的BeanDefinition集合
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        // 扫描符合条件的候选组件
        // 例如:AnnotationBeanNameGenerator#postProcessBeanDefinitionRegistry
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            // 解析通用注解@Lazy、@Primary、@DependsOn、@Role、@Description
            if (candidate instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                definitionHolder =
                    AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                // 将BeanDefinition添加到BeanFactory中
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}

/**
 * 扫描指定的class path，返回符合条件的BeanDefinition集合
 */
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    }
    else {
        return scanCandidateComponents(basePackage);
    }
}

private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        // classpath*:下的**/*.class
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        // 使用PathMatchingResourcePatternResolver
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
        boolean traceEnabled = logger.isTraceEnabled();
        boolean debugEnabled = logger.isDebugEnabled();
        for (Resource resource : resources) {
            if (resource.isReadable()) {
                try {
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                    if (isCandidateComponent(metadataReader)) {
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                        sbd.setSource(resource);
                        // 判断当前BeanDefinition是否是符合候选条件
                        if (isCandidateComponent(sbd)) {
                            candidates.add(sbd);
                        }
                    }
                }
                catch (Throwable ex) {
                    // exception
                }
            }
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
    }
    return candidates;
}
```



### 基础容器BeanFactory

基础容器BeanFactory都是由应用上下文ApplicationContext创建的

#### 类图

![BeanFactory继承关系](https://upload-images.jianshu.io/upload_images/845143-c11e52d3b2159a22.png)

> 实现多接口是为了区分在 Spring 内部操作对象传递和转化时，对对象的数据访问所做的限制。 <!--这就是接口隔离原则-->
>
> -  [ListableBeanFactory 接口表示这些 Bean 是可列表的]()
> - [HierarchicalBeanFactory 表示的是这些 Bean 是有继承关系的]()，也就是每个 Bean 有可能有父 Bean。
> - [AutowireCapableBeanFactory 接口定义 Bean 的自动装配规则]()。
> - [最终的默认实现类是 DefaultListableBeanFactory]()

数据集合操作：

> 由于实现了AliasRegistry，所以BeanFactory可以存储BeanDefinitions
>
> 由于实现了SingletonBeanResgistry，所以BeanFactory可以存储SingletonObjects

#### BeanFactory启动流程

- 以xml配置启动为例

重点找refresh在哪里被调用

```java
public ClassPathXmlApplicationContext( String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
      super(parent);
      //设置资源加载的路径
      setConfigLocations(configLocations);
      if (refresh) {
      		refresh();
      }
}
```

- 以java注解配置为例

重点找refresh在哪里被调用

```java
public ConfigurableApplicationContext run(String... args){
  ...
    refreshContext(context);
  ...
}


private void refreshContext(ConfigurableApplicationContext context){
    refresh(context);
    ...
}
```



#### BeanFactory被创建流程

- AbstractApplicationContext#refresh() 方法中的obtainFreshBeanFactory()


```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
      ....
}
```

- AbstractApplicationContext#obtainFreshBeanFactory() 方法：用于创建一个新的 IoC容器 ，这个 IoC容器 就是DefaultListableBeanFactory对象。

  ```java
  protected ConfigurableListableBeanFactory obtainFreshBeanFactory() { 
      // 主要是通过该方法完成IoC容器的刷新 
      refreshBeanFactory(); 
      ConfigurableListableBeanFactory beanFactory = getBeanFactory(); 
      if (logger.isDebugEnabled()) { 
          logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory); 
      }
      return beanFactory; 
  }
  ```

- AbstractRefreshableApplicationContext#refreshBeanFactory() 方法：[创建BeanFactory, 加载 BeanDefinition 对象注册到IoC容器中]()

  ```java
  protected final void refreshBeanFactory() throws BeansException { 
      // 如果之前有IoC容器，则销毁 
      if (hasBeanFactory()) { 
          destroyBeans(); closeBeanFactory();
      }
      try {
          // 创建IoC容器，也就是DefaultListableBeanFactory 
          DefaultListableBeanFactory beanFactory = createBeanFactory();   //其实就是new DefaultListableBeanFactory
          beanFactory.setSerializationId(getId()); 
          customizeBeanFactory(beanFactory); 
          // 加载BeanDefinition对象，并注册到IoC容器中（重点!!!） 
          loadBeanDefinitions(beanFactory); 
          synchronized (this.beanFactoryMonitor) { 
              this.beanFactory = beanFactory; 
          } 
      }catch (IOException ex) {
          throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex); 
      }
  }
  ```





### 补充：Resource核心组件

Resource与Context如何建立联系：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2FCXZ99ns72lgF7LfvpJEqBcuDicnjeZPBd9vv8gPibx6sNWcHicgMbLLIg/640)

从上图可以看出，Context 是把资源的加载、解析和描述工作委托给了 ResourcePatternResolver 类来完成，把资源的加载、解析和资源的定义整合在一起便于其他组件使用。Core 组件中还有很多类似的方式。



### 补充：封装beanDefinitions和singleObjects数据集合

- BeanDefinitionRegistry 封装了beanDefinitions

  - ```java
    public interface BeanDefinitionRegistry extends AliasRegistry { 
        // 给定bean名称，注册一个新的bean定义 
        void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException; 
        /** 根据指定Bean名移除对应的Bean定义 */ 
        void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException; /** 根据指定bean名得到对应的Bean定义 */ 
        BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException; /** 查找，指定的Bean名是否包含Bean定义 */ 
        boolean containsBeanDefinition(String beanName); 
        String[] getBeanDefinitionNames();//返回本容器内所有注册的Bean定义名称 
        int getBeanDefinitionCount();//返回本容器内注册的Bean定义数目 
        boolean isBeanNameInUse(String beanName);//指定Bean名是否被注册过。 
    }
    ```

- SingletonBeanRegistry封装了singleObjects

  - ```java
    public class DefaultSingletonBeanRegistry implements SingletonBeanRegistry {
    	// K:BeanName
    	// V:Bean实例对象
    	private Map<String, Object> singletonObjects = new HashMap<String, Object>();
    
    	@Override
    	public Object getSingleton(String beanName) {
    		return this.singletonObjects.get(beanName);
    	}
    
    	@Override
    	public void addSingleton(String beanName, Object bean) {
    		this.singletonObjects.put(beanName, bean);
    	}
    
    }
    ```

    





