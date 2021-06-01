---
title: spring-03-DI
date: 2021-05-29 21:40:10
tags:
---

#### beanFactory.createBean(...)以及doCreateBean(...)

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

