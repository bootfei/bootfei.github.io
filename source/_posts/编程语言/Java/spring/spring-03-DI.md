---
title: spring-03-DI
date: 2021-05-29 21:40:10
tags:
---

### beanFactory.createBean和doCreateBean

创建Bean的过程如下(省略部分代码，只保留主要逻辑)：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
	RootBeanDefinition mbdToUse = mbd;
    // 从BeanDefinition Map中得到要创建的bean的类型
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
    	mbdToUse.setBeanClass(resolvedClass);
    }
    try {
        mbdToUse.prepareMethodOverrides();
    } catch (BeanDefinitionValidationException ex) {
        // 异常
    }
    try {
        // 第一次调用后置处理器，判断是否加代理，Spring Boot中没有任何处理
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    } catch (Throwable ex) {
        // 异常
    }
    try {
        // 创建Bean的真正过程
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    	return beanInstance;
    } catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        // 异常
    } catch (Throwable ex) {
        // 异常
    }
}
```



```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 第二次调用后置处理器
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 创建实例对象
    final Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // 第三次调用后置处理器，通过后置处理器合并BeanDefinition,拿到所有需要注入的属性
                // ApplicationListenerDetector: 将属于ApplicationListener的Bean放入到ApplicationContext上下文的applicationListener集合中
                // CommonAnnotationBeanPostProcessor: 将@Resource、@webServiceRef、@EJB修饰的属性放入到RootBeanDefinition中
                // AutowiredAnnotationBeanPostProcessor: 将@Autowired、@Value、JSR-330相关注解放入到RootBeanDefinition中
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                // 异常
            }
            mbd.postProcessed = true;
        }
    }
    // 循环依赖相关
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
      // 当支持循环依赖时，就会提前暴露自己(为了其他对象可以注入自己)，第四次调用后置处理器,实际在Spring Boot中没有处理任何事情
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 判断是否需要属性注入,若需要则注入则完成注入,第五、六次调用后置处理器
        populateBean(beanName, mbd, instanceWrapper);
        // 初始化Spring，进行第七、八次后置处理器的调用
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        //异常
    }
    // Register bean as disposable.
    try {
        // 第9次调用后置处理器
      // DestructionAwareBeanPostProcessor的requiresDestruction方法和postProcessBeforeDestruction方法，具体destroy过程见:
      // org.springframework.beans.factory.support.DisposableBeanAdapter.destroy()方法
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }catch (BeanDefinitionValidationException ex) {
        // 异常
    }
    return exposedObject;
}
```


### beanFactory.populateBean

`populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw)`方法完成属性注入(省略部分代码，只保留主要逻辑):

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // 任何实现postProcessAfterInstantiation的BeanPostProcessor都可以修改postProcessAfterInstantiation的返回值,例如:动态注入
    // 第五次调用后置处理器，判断是否需要属性注入,CommonAnnotationBeanPostProcessor、AutowiredAnnotationBeanPostProcessor、ImportAwareBeanPostProcessor都默认为true
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                // 调用后置处理器 判断是否需要属性注入，实现InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation就不会进行属性注入
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    return;
                }
            }
        }
    }

    // 开始属性注入
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
    // 自动注入的方式。。。省略
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        // 第六次调用后置处理器
        // 1.在ConfigurationClassPostProcessor.ImportAwareBeanPostProcessor.postProcessProperties中指定BeanFactory
        // 2.在CommonAnnotationBeanPostProcessor.postProcessProperties中注入@Resurce、@WebServiceRef、@EJB相关
        // 3.其会在AutowiredAnnotationBeanPostProcessor.postProcessProperties中进行属性注入@Autowired、@Value、@Inject注解
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 完成属性注入
                PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                pvs = pvsToUse;
            }
        }
    }
}
```

### beanFactory.initializeBean

`initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd)`方法会执行工厂相关回调方法、init方法等BeanPostProcessor后置处理器，具体如下:

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    } else {
        // 执行或者回调Aware相关接口:BeanNameAware、BeanClassLoaderAware、BeanFactoryAware，只有这三个
        invokeAwareMethods(beanName, bean);
    }
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 第七次调用后置处理器
		// 1.ApplicationContextAwareProcessor: 执行EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware相关回调
		// 2.ImportAwareBeanPostProcessor: 实现ImportAware接口的Bean
		// 3.BeanPostProcessorChecker:没有做任何事情
		// 4.CommonAnnotationBeanPostProcessor继承自CommonAnnotationBeanPostProcessor, 执行带@PostConstract注解的方法
		// 5.AutowiredAnnotationBeanPostProcessor:没有做任何事情
		// 6.ApplicationListenerDetector:将ApplicationListener添加到applicationContext中
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }
    try {
        // 先处理实现InitializingBean,然后处理xml的init-method方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable ex) {
        // 异常
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 第8次调用后置处理器
		// 1.ApplicationContextAwareProcessor、ImportAwareBeanPostProcessor、: 没有做任何事情
		// 2.BeanPostProcessorChecker: 一些日志
		// 3.CommonAnnotationBeanPostProcessor、AutowiredAnnotationBeanPostProcessor:没有做任何事情
		// 4.ApplicationListenerDetector: 将符合条件的Bean放入到ApplicationContext.applicationListeners中
		// AOP代理 AnnotationAwareAspectJAutoProxyCreator
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

上述过程不仅仅是Bean的创建过程，还包括了创建Bean过程中相关的后置处理器逻辑，**Spring framework**相关的共6个后置处理器，具体见下文。