---
title: spring框架-04-bean包-Bean
date: 2022-10-17 19:52:16
categories: [java,spring]
tags:
---



# spring框架-04-bean包-Bean

![图片](https://mmbiz.qpic.cn/mmbiz_png/PW0wIHxgg3lvJAyXpj95WKrwovJQwiay1XP0WEEEPn0uuicicWbPcVo9picKjaErjr3psdE0jMY9Fqs8xoUA28rVmA/640)

### BeanDefinition（元信息的内存形式）

我们只是需要知道配置元信息被加载到内存之后是以BeanDefinition的形存在的即可。

### BeanDefinitionReader（元信息的读取）

<img src="https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2FtAJ3fVegLzicgADJvM2ibYFJKkuoS0ehty9TI437OtBRCE5u0Mo4VOfw/640" alt="图片" style="zoom:66%;" />

Spring中xml配置文件中一个个的Bean定义，但是Spring是要靠BeanDefinationReader了。

- 读取xml配置元信息，那么可以使用XmlBeanDefinationReader。

- 读取properties配置文件，那么可以使用PropertiesBeanDefinitionReader加载。

- 读取注解配置元信息，那么可以使用 AnnotatedBeanDefinitionReader加载。
- 读取自定义的配置信息，那么可以自定义BeanDefinationReader来控制配置元信息的加载

总的来说，BeanDefinationReader的作用就是加载配置元信息，并将其转化为内存形式的BeanDefination，存在某一个地方

### BeanDefinationRegistry（元信息内存形式的内存位置）

Spring通过BeanDefinationReader将配置元信息加载到内存生成相应的BeanDefination之后，就将其注册到BeanDefinationRegistry中，BeanDefinationRegistry就是一个存放BeanDefination的Map，通过特定的Bean定义的id，映射到相应的BeanDefination。

### BeanFactoryPostProcessor(解析BeanDef中的占位符)

BeanFactoryPostProcessor是容器启动阶段Spring提供的一个扩展点，主要负责对注册到BeanDefinationRegistry中的一个个的BeanDefination进行一定程度上的修改与替换。

例如我们的配置元信息中有些可能会修改的配置信息散落到各处，不够灵活，修改相应配置的时候比较麻烦，这时我们可以使用占位符的方式来配置。例如配置Jdbc的DataSource连接的时候可以这样配置:

```
<bean id="dataSource"  
    class="org.apache.commons.dbcp.BasicDataSource"  
    destroy-method="close">  
    <property name="maxIdle" value="${jdbc.maxIdle}"></property>  
    <property name="maxActive" value="${jdbc.maxActive}"></property>  
    <property name="maxWait" value="${jdbc.maxWait}"></property>  
    <property name="minIdle" value="${jdbc.minIdle}"></property>  
  
    <property name="driverClassName"  
        value="${jdbc.driverClassName}">  
    </property>  
    <property name="url" value="${jdbc.url}"></property>  
  
    <property name="username" value="${jdbc.username}"></property>  
    <property name="password" value="${jdbc.password}"></property>  
</bean> 
```

BeanFactoryPostProcessor就会对注册到BeanDefinationRegistry中的BeanDefination做最后的修改，替换`$`占位符为配置文件中的真实的数据。

至此，整个容器启动阶段就算完成了，容器的启动阶段的最终产物就是注册到BeanDefinationRegistry中的一个个BeanDefination了，这就是Spring为Bean实例化所做的预热的工作。<img src="https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XBzOtg5OQ6icxfKTvViaePJOczt8icmDz64YEmm1NjOzOkpl7C4dU8Arrc4xFFniayWje5TkK8AydcAtg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1" alt="Image" style="zoom:67%;" />





# Bean实例化阶段源码解析

Bean的创建过程

1. 验证，Merge BeanDefinition(`@Lookup`等)，判断是否是抽象类、单例、懒加载、FactoryBean等等
2. 推断构造方法
3. new对象(反射),放到Spring容器中
4. 缓存注解信息，并合并BeanDefination对象
5. 提前暴露自己(bean的半成品)
6. 判定是否需要属性注入
7. 完成属性注入
8. 调用生命周期回调方法`Aware`等
9. 完成代理AOP
10. put容器
11. 销毁对象



## context包高级容器入口：AbstractApplicationContext.refresh() 

->finishBeanFactoryInitialization(beanFactory)



## bean包基础容器入口：

BeanDefinition的解析入口：

##### AbstractBeanFactory.preInstantiateSingletons()

-> // Instantiate all remaining (non-lazy-init) singletons.

```
beanFactory.preInstantiateSingletons();
```

##### AbstractBeanFactory#getBean(beanName)

-> // Trigger initialization of all non-lazy singleton beans...

```java
public Object getBean(String name) throws BeansException {
  return this.doGetBean(name, (Class)null, (Object[])null, false);
}
```

##### AbstractBeanFactory#doGetBean(beanName)=》从BeanDefinition获取Bean的元数据

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    //TODO 验证Bean的名字是否非法
    final String beanName = transformedBeanName(name);
    Object bean;
	// 从单例缓存中检查是否存在单例bean，第一次必定返回null
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
    	bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}else {
        // 如果当前bean正在被创建则抛出异常，prototype bean
        if (isPrototypeCurrentlyInCreation(beanName)) {
        	throw new BeanCurrentlyInCreationException(beanName);
		}
        try {
            //TODO 解析合并后的BeanDefinition对象 待研究
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);
            // 确保当前Bean所依赖的Bean的初始化, @DependsOn
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    registerDependentBean(dep, beanName);
                    // 省略
                    // 实例化该例DependsOn的类
                    getBean(dep);  
                }
            }
            // 判断类是否是单例，prototype bean Spring是不会实例化的
			// SpringBoot和Spring MVC中@Controller、@Service、@Component、@Repository默认都是单例
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        // 创建bean的过程,对象的创建以及代理
                        return createBean(beanName, mbd, args);
                    } catch (BeansException ex) {
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
        } catch (BeansException ex) {
            // Bean创建失败，清除Bean创建标识
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }
	return (T) bean;
}
```

## AbstractBeanFactory#doCreateBean(beanName)

##### AbstractBeanFactory#createBeanInstance：实例化

其实也就是 调用对象的构造方法实例化对象  ----> new ServiceA()

##### AbstractBeanFactory#populateBean：填充属性

这一步主要是多bean的依赖属性进行填充

##### AbstractBeanFactory#initializeBean：调用初始化方法

调用spring xml中的init() 方法。
