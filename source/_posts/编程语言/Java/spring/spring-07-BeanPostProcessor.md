---
title: spring-07-BeanPostProcessor
date: 2021-06-02 11:31:13
tags:
---

Orgin: https://www.effiu.cn/blog/2020/10/27/spring/3_beanPostProcessor/

**Spring**的`BeanPostProcessor`后置处理器承担了很多工作

- AOP、代理
- 构造方法注入的构造方法推断，构造方法上使用`@Autowired`等注解时
- 循环依赖，提前暴露未完成的Bean
- 依赖注入，`@Resource`、`@Autowired`、`@Value`等，不同注解依赖不同的后置处理器
-  动态注入
- 执行`Aware`回调
- `@Import`注解，通过`ImportAwareBeanPostProcessor`将`@Import`修饰的注解注入到`@Import`指定的类中
- 初始化和销毁方法，`@PostConsturct`、`@PreDestroy`后置处理完成
- `ApplicationListener`的监听等



## 相关BeanPostProcessor

![ ](https://images.effiu.cn/blog/spring/BeanPostProcessor.png)

在**Spring Boot**中**Spring framework**相关**BeanPostProcesso**如下:

1. ApplicationContextAwareProcessor
2. ConfigurationClassPostProcessor.ImportAwareBeanPostProcessor
3. PostProcessorRegistrationDelegate.BeanPostProcessorChecker
4. CommonAnnotationBeanPostProcessor
5. AutowiredAnnotationBeanPostProcessor
6. ApplicationListenerDetector

> 忽略Spring MVC、Spring Boot、Spring Cloud相关BeanPostProcessor，例如:Spring Boot的`ConfigurationPropertiesBindingPostProcessor`,Spring Cloud的`ConfigurationPropertiesBeans`等等。

上述6个后置处理器相关接口以及发挥作用的方法(标红的):

![BeanPostProcessor类图](https://images.effiu.cn/blog/spring/BeanPostProcessor_class.jpg)

> 上述标红的为真正执行的类,类中的成员仅仅是一些重要的**field**和方法，即在获取Bean过程中执行后置处理器相关的方法。

## 相关源码

> **BeanPostProcessor**在`org.springframework.context.support.AbstractApplicationContext#refresh`中的`obtainFreshBeanFactory()`和`registerBeanPostProcessors()`方法中注册的

`org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory`加载了`ApplicationContextAwareProcessor`和`ApplicationListenerDetector`两个后置处理器

```
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 第一个BeanPostProcessor:ApplicationContextAwareProcessor
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    // 配置自动注入忽略的接口，需要以其他的方式完成相关Bean的引用,例如实现ApplicationContextAware接口完成引用
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // 自动装配时指定实现类(一个接口有多个实现类的情况),作用类似于@Primary注解
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    // 第二个BeanPostProcessor
	// ApplicationListenerDetector用于将ApplicationListener子类Bean放入到{@link applicationListeners} 集合中
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // ...省略

    // Register default environment beans. 三个Bean，environment、systemProperties、systemEnvironment, 可以直接通过@Autowried等注解使用
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

`org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors`通过调用`ConfigurationClassPostProcessor#postProcessBeanFactory()`方法注册了`ImportAwareBeanPostProcessor`后置处理器。

```
// ConfigurationClassPostProcessor部分源码
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    int factoryId = System.identityHashCode(beanFactory);
    if (this.factoriesPostProcessed.contains(factoryId)) {
        throw new IllegalStateException(
        "postProcessBeanFactory already called on this post-processor against " + beanFactory);
    }
    this.factoriesPostProcessed.add(factoryId);
    if (!this.registriesPostProcessed.contains(factoryId)) {
        // BeanDefinitionRegistryPostProcessor hook apparently not supported...
        // Simply call processConfigurationClasses lazily at this point then.
        processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
    }

    enhanceConfigurationClasses(beanFactory);
    beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}
org.springframework.context.support.AbstractApplicationContext#registerBeanPostProcessors`注册了`BeanPostProcessorChecker`、`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`、以及重新注册了`ApplicationListenerDetector
public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
    // 遍历beanDefinitionNames获取BeanPostProcessorNames
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    // BeanPostProcessorChecker用于当在BeanPostProcessor实例化期间创建Bean时，当某个Bean不适合所有BeanPostProcessor时，记录信息. BeanPostProcessorChecker内部会比较beanProcessorTargetCount与beanFactory.getBeanPostProcessorCount()
    beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
    // Separate between BeanPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            // CommonAnnotationBeanPostProcessor、AutowiredAnnotationBeanPostProcessor两个BeanPostProcessor
            priorityOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // First, register the BeanPostProcessors that implement PriorityOrdered.
    // 注册实现PriorityOrdered的BeanPostProcessors
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

    // Next, register the BeanPostProcessors that implement Ordered.
    // 注册实现Ordered的BeanPostProcessors
    List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
    for (String ppName : orderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        orderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, orderedPostProcessors);

    // Now, register all regular BeanPostProcessors.
    // 注册所有常规的BeanPostProcessors
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String ppName : nonOrderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        nonOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

    // Finally, re-register all internal BeanPostProcessors.
    // 最后重新注册所有内部BeanPostProcessor
    sortPostProcessors(internalPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, internalPostProcessors);

    // Re-register post-processor for detecting inner beans as ApplicationListeners,
    // moving it to the end of the processor chain (for picking up proxies etc).
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

## BeanPostProcessor作用

排序后的BeanPostProcessor顺序为:

1. ApplicationContextAwareProcessor
2. ConfigurationClassPostProcessor.ImportAwareBeanPostProcessor
3. PostProcessorRegistrationDelegate.BeanPostProcessorChecker
4. CommonAnnotationBeanPostProcessor
5. AutowiredAnnotationBeanPostProcessor
6. ApplicationListenerDetector

### ApplicationContextAwareProcessor

```
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
          bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
          bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
        return bean;
    }

    AccessControlContext acc = null;

    if (System.getSecurityManager() != null) {
        acc = this.applicationContext.getBeanFactory().getAccessControlContext();
    }

    if (acc != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareInterfaces(bean);
            return null;
        }, acc);
    }
    else {
        invokeAwareInterfaces(bean);
    }

    return bean;
}

private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof EnvironmentAware) {
        ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof EmbeddedValueResolverAware) {
        ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
    }
    if (bean instanceof ResourceLoaderAware) {
        ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
    }
    if (bean instanceof ApplicationEventPublisherAware) {
        ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
    }
    if (bean instanceof MessageSourceAware) {
        ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
    }
    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
}
```

`ApplicationContextAwareProcessor`加载的是`EnvironmentAware`、`EmbeddedValueResolverAware`、`ResourceLoaderAware`、`ApplicationEventPublisherAware`、`MessageSourceAware`、`ApplicationContextAware`6个用于回调的Aware。

### ImportAwareBeanPostProcessor

> 是`ConfigurationClassPostProcessor`的子类，在Spring容器启动过程中，由其父类完成注册

```
private static class ImportAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter {

    private final BeanFactory beanFactory;

    public ImportAwareBeanPostProcessor(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    @Override
    public PropertyValues postProcessProperties(@Nullable PropertyValues pvs, Object bean, String beanName) {
        // 在AutowiredAnnotationBeanPostProcessor的postProcessProperties方法注入Bean之前,注入BeanFactory,
        if (bean instanceof EnhancedConfiguration) {
            ((EnhancedConfiguration) bean).setBeanFactory(this.beanFactory);
        }
        return pvs;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // 若该bean是ImportAware, 则将@Import注解的类
        // (支持@Configuration、@Component,实现ImportSelector、ImportBeanDefinitionRegistrar接口的实现类)
        // 通过ImportAware的setImportMetadata将该注解注入到@Import指定类中
        if (bean instanceof ImportAware) {
            ImportRegistry ir = this.beanFactory.getBean(IMPORT_REGISTRY_BEAN_NAME, ImportRegistry.class);
            AnnotationMetadata importingClass = ir.getImportingClassFor(ClassUtils.getUserClass(bean).getName());
            if (importingClass != null) {
                ((ImportAware) bean).setImportMetadata(importingClass);
            }
        }
        return bean;
    }
}
```

### BeanPostProcessorChecker

> 没有实际逻辑，只是用于在BeanPostProcessor实例化期间创建Bean时记录日志。

### CommonAnnotationBeanPostProcessor

`CommonAnnotationBeanPostProcessor`支持的注解,`@Resource`、`@EJB`、`@WebServiceRef`，其继承了`InitDestroyAnnotationBeanPostProcessor`，`InitDestroyAnnotationBeanPostProcessor`支持`@PostConstruct`和`@PreDestroy`注解。

```
@Nullable
private static final Class<? extends Annotation> webServiceRefClass;
@Nullable
private static final Class<? extends Annotation> ejbClass;
private static final Set<Class<? extends Annotation>> resourceAnnotationTypes = new LinkedHashSet<>(4);

static {
    webServiceRefClass = loadAnnotationType("javax.xml.ws.WebServiceRef");
    ejbClass = loadAnnotationType("javax.ejb.EJB");
    // 配置支持的注解
    resourceAnnotationTypes.add(Resource.class);
    if (webServiceRefClass != null) {
        resourceAnnotationTypes.add(webServiceRefClass);
    }
    if (ejbClass != null) {
        resourceAnnotationTypes.add(ejbClass);
    }
}
// 将支持的注解解析为InjectionMetadata,放入到injectionMetadataCache
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    super.postProcessMergedBeanDefinition(beanDefinition, beanType, beanName);
    InjectionMetadata metadata = findResourceMetadata(beanName, beanType, null);
    metadata.checkConfigMembers(beanDefinition);
}

// 从injectionMetadataCache中取得InjectionMetadata，然后完成注入
@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass(), pvs);
    try {
        // 属性注入
        metadata.inject(bean, beanName, pvs);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
    }
    return pvs;
}
```

### InjectionMetadata

`InjectionMetadata`要注入的数据元，`InjectElement`是其内部类。`MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition`方法会将BeanPostProcessor支持的注解解析为`InjectionMetadata`，然后由`InstantiationAwareBeanPostProcessorAdapter.postProcessProperties`方法完成注入。

`InjectionMetadata`部分源码如下:

```
// 待注入的元素,field或者method
private final Collection<InjectedElement> injectedElements;

public abstract static class InjectedElement {
	protected final Member member;
	protected final boolean isField;
}
// postProcessProperties中会调用该方法
protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs)
				throws Throwable {
    if (this.isField) {
        Field field = (Field) this.member;
        ReflectionUtils.makeAccessible(field);
        field.set(target, getResourceToInject(target, requestingBeanName));
    }
    else {
        if (checkPropertySkipping(pvs)) {
            return;
        }
        try {
            Method method = (Method) this.member;
            ReflectionUtils.makeAccessible(method);
            method.invoke(target, getResourceToInject(target, requestingBeanName));
        }
        catch (InvocationTargetException ex) {
            throw ex.getTargetException();
        }
    }
}
```

### AutowiredAnnotationBeanPostProcessor

`AutowiredAnnotationBeanPostProcessor`支持`@Autowired`、`@Value`、`@Inject`、`@Lookup`等注解

> `AutowiredAnnotationBeanPostProcessor`会将支持的注解的field或者method解析为`InjectMetadata`，但是`AutowiredAnnotationBeanPostProcessor`内部自己实现了`InjectedElement`的子类`AutowiredFieldElement`和`AutowiredMethodElement`，分别用于属性注入和方法注入。

```
public AutowiredAnnotationBeanPostProcessor() {
    // 配置支持@Autowired、@Value以及JSR-330相关注解
    this.autowiredAnnotationTypes.add(Autowired.class);
    this.autowiredAnnotationTypes.add(Value.class);
    try {
        this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
                                          ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
    }
    catch (ClassNotFoundException ex) {
        // JSR-330 API not available - simply skip.
    }
}

// ··· 省略

// 将支持的注解解析为InjectionMetadata(AutowiredFieldElement、AutowiredMethodElement)
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
    metadata.checkConfigMembers(beanDefinition);
}

@Override
@Nullable
public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName)
    throws BeanCreationException {

    // Let's check for lookup methods here...
    if (!this.lookupMethodsChecked.contains(beanName)) {
        if (AnnotationUtils.isCandidateClass(beanClass, Lookup.class)) {
            try {
                Class<?> targetClass = beanClass;
                do {
                    ReflectionUtils.doWithLocalMethods(targetClass, method -> {
                        Lookup lookup = method.getAnnotation(Lookup.class);
                        if (lookup != null) {
                            Assert.state(this.beanFactory != null, "No BeanFactory available");
                            LookupOverride override = new LookupOverride(method, lookup.value());
                            try {
                                RootBeanDefinition mbd = (RootBeanDefinition)
                                    this.beanFactory.getMergedBeanDefinition(beanName);
                                mbd.getMethodOverrides().addOverride(override);
                            }
                            catch (NoSuchBeanDefinitionException ex) {
                                throw new BeanCreationException(beanName,
                                                                "Cannot apply @Lookup to beans without corresponding bean definition");
                            }
                        }
                    });
                    targetClass = targetClass.getSuperclass();
                }
                while (targetClass != null && targetClass != Object.class);

            }
            catch (IllegalStateException ex) {
                throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
            }
        }
        this.lookupMethodsChecked.add(beanName);
    }

    // Quick check on the concurrent map first, with minimal locking.
    Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
    if (candidateConstructors == null) {
        // Fully synchronized resolution now...
        synchronized (this.candidateConstructorsCache) {
            candidateConstructors = this.candidateConstructorsCache.get(beanClass);
            if (candidateConstructors == null) {
                Constructor<?>[] rawCandidates;
                try {
                    // 获取所有的构造方法
                    rawCandidates = beanClass.getDeclaredConstructors();
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(beanName,
                                                    "Resolution of declared constructors on bean Class [" + beanClass.getName() +
                                                    "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
                }
                List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
                Constructor<?> requiredConstructor = null;
                Constructor<?> defaultConstructor = null;
                Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);
                int nonSyntheticConstructors = 0;
                for (Constructor<?> candidate : rawCandidates) {
                    // 是否由编译器自己生成的构造方法
                    if (!candidate.isSynthetic()) {
                        nonSyntheticConstructors++;
                    }
                    else if (primaryConstructor != null) {
                        continue;
                    }
                    // 检查是否存在使用支持注解修饰的类构造方法
                    MergedAnnotation<?> ann = findAutowiredAnnotation(candidate);
                    if (ann == null) {
                        // 检查父类构造方法是否存在支持的注解
                        Class<?> userClass = ClassUtils.getUserClass(beanClass);
                        if (userClass != beanClass) {
                            try {
                                Constructor<?> superCtor =
                                    userClass.getDeclaredConstructor(candidate.getParameterTypes());
                                ann = findAutowiredAnnotation(superCtor);
                            }
                            catch (NoSuchMethodException ex) {
                                // Simply proceed, no equivalent superclass constructor found...
                            }
                        }
                    }
                    if (ann != null) {
                        // 只能存在一个需要注入的构造方法，否则会抛出异常
                        if (requiredConstructor != null) {
                            throw new BeanCreationException(beanName,
                                                            "Invalid autowire-marked constructor: " + candidate +
                                                            ". Found constructor with 'required' Autowired annotation already: " +
                                                            requiredConstructor);
                        }
                        // 判断是否需要注入，若需要那么当不存在要注入的bean时会抛出异常
                        boolean required = determineRequiredStatus(ann);
                        if (required) {
                            if (!candidates.isEmpty()) {
                                throw new BeanCreationException(beanName,
                                                                "Invalid autowire-marked constructors: " + candidates +
                                                                ". Found constructor with 'required' Autowired annotation: " +
                                                                candidate);
                            }
                            requiredConstructor = candidate;
                        }
                        candidates.add(candidate);
                    }
                    else if (candidate.getParameterCount() == 0) {
                        defaultConstructor = candidate;
                    }
                }
                if (!candidates.isEmpty()) {
                    // Add default constructor to list of optional constructors, as fallback.
                    if (requiredConstructor == null) {
                        if (defaultConstructor != null) {
                            candidates.add(defaultConstructor);
                        }
                        else if (candidates.size() == 1 && logger.isInfoEnabled()) {
                            logger.info("Inconsistent constructor declaration on bean with name '" + beanName +
                                        "': single autowire-marked constructor flagged as optional - " +
                                        "this constructor is effectively required since there is no " +
                                        "default constructor to fall back to: " + candidates.get(0));
                        }
                    }
                    candidateConstructors = candidates.toArray(new Constructor<?>[0]);
                }
                else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
                    candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
                }
                else if (nonSyntheticConstructors == 2 && primaryConstructor != null &&
                         defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
                    candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};
                }
                else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {
                    candidateConstructors = new Constructor<?>[] {primaryConstructor};
                }
                else {
                    candidateConstructors = new Constructor<?>[0];
                }
                this.candidateConstructorsCache.put(beanClass, candidateConstructors);
            }
        }
    }
    return (candidateConstructors.length > 0 ? candidateConstructors : null);
}

// 从injectionMetadataCache中取出需要出入的原始数据完成注入
@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        metadata.inject(bean, beanName, pvs);
    }
    catch (BeanCreationException ex) {
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```

### ApplicationListenerDetector

```
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType,
                                            String beanName) {
    // 若Bean是ApplicationListener则将其放入到singletonNames内
    // 在postProcessAfterInitialization方法中，将ApplicationListener子类放入到ApplicationListeners集合中
    if (ApplicationListener.class.isAssignableFrom(beanType)) {
        this.singletonNames.put(beanName, beanDefinition.isSingleton());
    }
}

@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) {
    return bean;
}

@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean instanceof ApplicationListener) {
        // potentially not detected as a listener by getBeanNamesForType retrieval
        // getBeanNamesForType可能不会将其判断为监听器,所以此处判断是否是ApplicationListener若是则将其加入到ApplicationListeners中
        Boolean flag = this.singletonNames.get(beanName);
        if (Boolean.TRUE.equals(flag)) {
            // singleton bean (top-level or inner): register on the fly
            this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
         } else if (Boolean.FALSE.equals(flag)) {
                this.singletonNames.remove(beanName);
         }
    }
    return bean;
}
```

## BeanPostProcessor的执行顺序

> `BeanPostProcessor`在**Bean**的实例化过程中每个方法执行的顺序。上文已经指出在**Bean**的实例化过程中8次调用后置处理器的地方。

### postProcessBeforeInstantiation

第一次调用后置处理器是`InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation`，其会在目标对象被实例化之前调用，因为此时目标对象未被实例化，所以可以返回自定义对象，比如代理对象。

`AnnotationAwareAspectJAutoProxyCreator`是`InstantiationAwareBeanPostProcessor`的一个子类，所有使用`@AspectJ`注解修饰的类，都会被其自动识别，其实现了`BeanFactoryAware`接口，所有持有beanFactory对象，会在创建代理类后将**targetClass**赋给代理类。

> `AnnotationAwareAspectJAutoProxyCreator`与AOP相关，待后续研究

### determineCandidateConstructors

第二次调用后置处理器是`SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors`方法，主要是推断创建Bean的构造方法。具体见[AutowiredAnnotationBeanPostProcessor](https://www.effiu.cn/blog/2020/10/27/spring/3_beanPostProcessor/#AutowiredAnnotationbeanPostProcessor)。

### postProcessMergedBeanDefinition

第三次调用后置处理器是`MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition`其主要作用的解析类中要注入的对象然后放入到`BeanDefinition`中，然后会在``InstantiationAwareBeanPostProcessor.postProcessProperties`中完成属性注入

### getEarlyBeanReference

第四次调用后置处理器是`SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference`，用于循环依赖中提前暴露Bean

### postProcessAfterInstantiation

第五次调用后置处理器是`InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation`主要是在Bean完成实例化后判断是否需要属性注入，返回false则不需要完成属性注入

### postProcessProperties

第六次调用后置处理器是`InstantiationAwareBeanPostProcessor.postProcessProperties`，主要是完成属性注入。

- 在``ConfigurationClassPostProcessor.ImportAwareBeanPostProcessor.postProcessProperties`中指定`BeanFactory`
- 在`CommonAnnotationBeanPostProcessor.postProcessProperties`中注入`@Resurce`、`@WebServiceRef`、`@EJB`、`@PostConstruct`、`@PreDestory`注解相关
- 在`AutowiredAnnotationBeanPostProcessor.postProcessProperties`中进行属性注入`@Autowired`、`@Value`、`@Inject`注解相关属性

### postProcessBeforeInitialization

第七次调用后置处理器是`BeanPostProcessor.postProcessBeforeInitialization`，`BeanPostProcessor`是最顶层接口，所以所有后置处理器都有该方法。存在逻辑的如下:

- `ApplicationContextAwareProcessor`: 执行`EnvironmentAware`、`EmbeddedValueResolverAware`、`ResourceLoaderAware`、`ApplicationEventPublisherAware`、`MessageSourceAware`、`ApplicationContextAware`相关回调
- `ImportAwareBeanPostProcessor`: 实现`ImportAware`接口的Bean
- `CommonAnnotationBeanPostProcessor`其继承了`InitDestroyAnnotationBeanPostProcessor`类，其会执行`@PostConstruct`方法

### postProcessAfterInitialization

第八次调用后置处理器是`BeanPostProcessor.postProcessAfterInitialization`，`BeanPostProcessor`是最顶层接口，所以所有后置处理器都有该方法。

- `ApplicationListenerDetector`，将符合条件的**Bean**放入到`ApplicationContext.applicationListeners`中，用于监听**Application**事件
-  `AnnotationAwareAspectJAutoProxyCreator`，待研究

### requiresDestruction和postProcessBeforeDestruction

第9次调用后置处理器，`DestructionAwareBeanPostProcessor#requiresDestruction`和`DestructionAwareBeanPostProcessor#postProcessBeforeDestruction`，判断Bean实例是否需要该后置处理器销毁。

```
public DisposableBeanAdapter(Object bean, String beanName, RootBeanDefinition beanDefinition,
			List<BeanPostProcessor> postProcessors, @Nullable AccessControlContext acc) {
    // 构造方法中返回该bean在销毁时需要执行的后置处理器,filterPostProcessors
    this.beanPostProcessors = filterPostProcessors(postProcessors, bean);
}

// DestructionAwareBeanPostProcessor#requiresDestruction方法
private List<DestructionAwareBeanPostProcessor> filterPostProcessors(List<BeanPostProcessor> processors, Object bean) {
    List<DestructionAwareBeanPostProcessor> filteredPostProcessors = null;
    if (!CollectionUtils.isEmpty(processors)) {
        filteredPostProcessors = new ArrayList<>(processors.size());
        for (BeanPostProcessor processor : processors) {
            if (processor instanceof DestructionAwareBeanPostProcessor) {
                DestructionAwareBeanPostProcessor dabpp = (DestructionAwareBeanPostProcessor) processor;
                if (dabpp.requiresDestruction(bean)) {
                    filteredPostProcessors.add(dabpp);
                }
            }
        }
    }
    return filteredPostProcessors;
}

// 销毁调用方法
public void destroy() {
    // 处理@PreDestroy注解
    if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
        for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
            processor.postProcessBeforeDestruction(this.bean, this.beanName);
        }
    }
	// DisposableBean接口
    if (this.invokeDisposableBean) {
        try {
            // ··· 省略
            ((DisposableBean) this.bean).destroy();
        }
        catch (Throwable ex) {
            // throw ex
        }
    }
	// 推断的destroyMethod，例如close、shutdown方法,xml中的destroy-method、default-destroy-method属性
    if (this.destroyMethod != null) {
        invokeCustomDestroyMethod(this.destroyMethod);
    }
    else if (this.destroyMethodName != null) {
        Method methodToInvoke = determineDestroyMethod(this.destroyMethodName);
        if (methodToInvoke != null) {
            invokeCustomDestroyMethod(ClassUtils.getInterfaceMethodIfPossible(methodToInvoke));
        }
    }
}
```

当销毁Bean时:

1. `org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroySingletons`
2. `org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroySingleton`
3. `org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#destroyBean`

> destroyBean的源码如下:

```
protected void destroyBean(String beanName, @Nullable DisposableBean bean) {
    // Trigger destruction of dependent beans first...
    Set<String> dependencies;
    synchronized (this.dependentBeanMap) {
        // Within full synchronization in order to guarantee a disconnected Set
        dependencies = this.dependentBeanMap.remove(beanName);
    }
    // 销毁依赖的bean
    if (dependencies != null) {
        for (String dependentBeanName : dependencies) {
            destroySingleton(dependentBeanName);
        }
    }

    // Actually destroy the bean now...
    if (bean != null) {
        try {
            // 此处调用的是DisposableBeanAdapter.destroy()方法。
            bean.destroy();
        }
        catch (Throwable ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Destruction of bean with name '" + beanName + "' threw an exception", ex);
            }
        }
    }

    // Trigger destruction of contained beans...
    Set<String> containedBeans;
    synchronized (this.containedBeanMap) {
        // Within full synchronization in order to guarantee a disconnected Set
        containedBeans = this.containedBeanMap.remove(beanName);
    }
    if (containedBeans != null) {
        for (String containedBeanName : containedBeans) {
            destroySingleton(containedBeanName);
        }
    }

    // Remove destroyed bean from other beans' dependencies.
    synchronized (this.dependentBeanMap) {
        for (Iterator<Map.Entry<String, Set<String>>> it = this.dependentBeanMap.entrySet().iterator(); it.hasNext();) {
            Map.Entry<String, Set<String>> entry = it.next();
            Set<String> dependenciesToClean = entry.getValue();
            dependenciesToClean.remove(beanName);
            if (dependenciesToClean.isEmpty()) {
                it.remove();
            }
        }
    }

    // Remove destroyed bean's prepared dependency information.
    this.dependenciesForBeanMap.remove(beanName);
}
```