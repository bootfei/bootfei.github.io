---
title: spring框架-04-bean包-Bean
date: 2020-10-17 19:52:16
categories: [java,spring]
tags:
---



## Bean实例的创建

![图片](https://mmbiz.qpic.cn/mmbiz_png/PW0wIHxgg3lvJAyXpj95WKrwovJQwiay1XP0WEEEPn0uuicicWbPcVo9picKjaErjr3psdE0jMY9Fqs8xoUA28rVmA/640)

### BeanDefinition（元信息的内存形式）

我们只是需要知道配置元信息被加载到内存之后是以BeanDefinition的形存在的即可。

### BeanDefinitionReader（元信息的读取）

<img src="https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2FtAJ3fVegLzicgADJvM2ibYFJKkuoS0ehty9TI437OtBRCE5u0Mo4VOfw/640" alt="图片" style="zoom:66%;" />

Spring中xml配置文件中一个个的Bean定义，但是Spring是要靠BeanDefinationReader了。

不同的BeanDefinationReader各自拥有各自的本领。

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



### Bean实例化阶段

需要指出，容器启动阶段与Bean实例化阶段存在多少时间差，Spring把这个决定权交给了我们程序员

- 如果选择懒汉式的方式，那么直到我们伸手向Spring要依赖对象实例之前，其都是以BeanDefinationRegistry中的一个个的BeanDefination的形式存在，也就是Spring只有在我们需要依赖对象的时候才开启相应对象的实例化阶段。
- 如果选择饿汉式的方式，容器启动阶段完成之后，将立即启动Bean实例化阶段，通过隐式的调用所有依赖对象的getBean方法来实例化所有配置的Bean并保存起来。 <!--其实饿汉式也是通过懒汉式实现的-->

#### Bean对象创建策略

对象的创建采用了[策略模式]()，借助我们前面BeanDefinationRegistry中的BeanDefination,我们可以使用[反射的方式]()创建对象，也可以使用CGlib字节码生成创建对象。这个时候，内存中应该已经有一个我们想要的具体的依赖对象的实例了。

#### BeanWrapper——对象的外衣

Spring中的Bean并不是单独存在的，由于Spring IOC容器中要管理多种类型的对象，因此为了统一对不同类型对象的访问，***Spring给所有创建的Bean实例穿上了一层外套***，这个外套就是BeanWrapper。BeanWrapper实际上是对反射相关API的简单封装，使得上层使用反射完成相关的业务逻辑大大的简化，我

#### 设置Bean对象属性

上一步包裹在BeanWrapper中的对象，Spring需要为其设置属性以及依赖对象。

- 对于基本类型的属性
  - 如果配置元信息中有配置，那么将直接使用配置元信息中的设置值赋值即可，
  - 如果没有配置，那么得益于JVM对象实例化过程，属性依然可以被赋予默认的初始化零值。
- 对于引用类型的属性，Spring会将所有已经创建好的对象放入一个Map结构中，此时Spring会检查所依赖的对象是否已经在容器的map中有对应对象的实例了。
  - 如果有，那么直接注入
  - 如果没有，那么Spring会暂时放下该对象的实例化过程，转而先去实例化依赖对象，再回过头来完成该对象的实例化过程。

> 这里有一个Spring中的经典问题，那就是Spring是如何解决循环依赖的？
>
> 这里简单提一下，Spring是通过三级缓存解决循环依赖，并且只能解决Setter注入的循环依赖

#### 检查Aware相关接口

如果想要依赖Spring中的相关对象，使用Spring的相关API,那么可以实现相应的Aware接口，Spring IOC容器就会为我们自动注入相关依赖对象实例。

对于BeanFactory来说，这一步的实现是先检查相关的Aware接口，然后去Spring的对象池(也就是容器的Map结构)中去查找相关的实例(例如对于ApplicationContextAware接口，就去找ApplicationContext实例)，也就是说我们必须要在配置文件中或者使用注解的方式，将相关实例注册容器中，BeanFactory才可以为我们自动注入。

而对于ApplicationContext，由于其本身继承了一系列的相关接口，所以当检测到Aware相关接口，需要相关依赖对象的时候，ApplicationContext完全可以将自身注入到其中，ApplicationContext实现这一步是通过下面BeanPostProcessor。

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c34631d663f5498e8b151a0eaf5905fe~tplv-k3u1fbpfcp-watermark.image)

> 例如ApplicationContext继承自ResourceLoader和MessageSource，那么当我们实现ResourceLoaderAware和MessageSourceAware相关接口时，就将其自身注入到业务对象中即可。

#### BeanPostProcessor前置处理

> 只要记住BeanFactoryPostProcessor存在于容器启动阶段，而BeanPostProcessor存在于对象实例化阶段，BeanFactoryPostProcessor关注***对象被创建之前*** 那些配置的修改，而BeanPostProcessor阶段关注***对象已经被创建之后*** 的功能增强，替换等操作

BeanPostProcessor与BeanFactoryPostProcessor都是Spring在Bean生产过程中强有力的扩展点。如果你还对它感到很陌生，那么你肯定知道Spring中著名的AOP(面向切面编程)，其实就是依赖BeanPostProcessor对Bean对象功能增强的。

BeanPostProcessor前置处理就是在要生产的Bean实例放到容器之前，允许程序员对Bean实例进行一定程度的修改，替换等操作。

前面讲到的ApplicationContext对于Aware接口的检查与自动注入就是通过BeanPostProcessor实现的，在这一步Spring将检查Bean中是否实现了相关的Aware接口，如果是的话，那么就将其自身注入Bean中即可。Spring中AOP就是在这一步实现的偷梁换柱，产生对于原生对象的代理对象，然后将对原生对象上的方法调用，转而使用代理对象的相同方法调用实现的。

#### 自定义初始化逻辑

在所有的准备工作完成之后，如果我们的Bean还有一定的初始化逻辑，那么Spring将允许我们通过两种方式配置我们的初始化逻辑：(1)InitializingBean  (2)配置init-method参数

一般通过配置init-method方法比较灵活。

#### BeanPostProcess后置处理

与前置处理类似，这里是在Bean自定义逻辑也执行完成之后，Spring又留给我们的最后一个扩展点。我们可以在这里在做一些我们想要的扩展。

#### 自定义销毁逻辑

这一步对应自定义初始化逻辑，同样有两种方式：(1)实现DisposableBean接口 (2)配置destory-method参数。

这里一个比较典型的应用就是配置dataSource的时候destory-method为数据库连接的close()方法。

#### 使用

经过了以上道道工序，我们像对待平常的对象一样对待Spring为我们产生的Bean实例

#### 调用回调销毁接口

Spring的Bean在为我们服务完之后，马上就要消亡了(通常是在容器关闭的时候)，这时候Spring将以回调的方式调用我们自定义的销毁逻辑，



![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a23065fc2124b79993b669044da8e82~tplv-k3u1fbpfcp-watermark.image)

> 需要指出，容器启动阶段与Bean实例化阶段之间的桥梁就是我们可以选择自定义配置的延迟加载策略，如果我们配置了Bean的延迟加载策略，那么只有我们在真实的使用依赖对象的时候，Spring才会开始Bean的实例化阶段。而如果我们没有开启Bean的延迟加载，那么在容器启动阶段之后，就会紧接着进入Bean实例化阶段，通过隐式的调用getBean方法，来实例化相关Bean。



### Bean实例化阶段源码解析

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



#### 高级容器refresh()中finishBeanFactoryInitialization(beanFactory)

注意事项：Bean的IoC、DI和AOP都是发生在此步骤

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```

#### beanFactory.preInstantiateSingletons()

实例化剩余的单例bean（非懒加载方式）

```java
public void preInstantiateSingletons() throws BeansException {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Pre-instantiating singletons in " + this);
        }

        List<String> beanNames = new ArrayList(this.beanDefinitionNames);
        Iterator var2 = beanNames.iterator();

        while(true) {
            String beanName;
            Object bean;
            do {
                while(true) {
                    RootBeanDefinition bd;
                    do {
                        do {
                            do {
                                if (!var2.hasNext()) {
                                    var2 = beanNames.iterator();

                                    while(var2.hasNext()) {
                                        beanName = (String)var2.next();
                                        Object singletonInstance = this.getSingleton(beanName);
                                        if (singletonInstance instanceof SmartInitializingSingleton) {
                                            SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton)singletonInstance;
                                            if (System.getSecurityManager() != null) {
                                                AccessController.doPrivileged(() -> {
                                                    smartSingleton.afterSingletonsInstantiated();
                                                    return null;
                                                }, this.getAccessControlContext());
                                            } else {
                                                smartSingleton.afterSingletonsInstantiated();
                                            }
                                        }
                                    }

                                    return;
                                }

                                beanName = (String)var2.next();
                                bd = this.getMergedLocalBeanDefinition(beanName);
                            } while(bd.isAbstract());
                        } while(!bd.isSingleton());
                    } while(bd.isLazyInit());

                    if (this.isFactoryBean(beanName)) {
                        bean = this.getBean("&" + beanName);
                        break;
                    }

                    this.getBean(beanName);
                }
            } while(!(bean instanceof FactoryBean));

            FactoryBean<?> factory = (FactoryBean)bean;
            boolean isEagerInit;
            if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                SmartFactoryBean var10000 = (SmartFactoryBean)factory;
                ((SmartFactoryBean)factory).getClass();
                isEagerInit = (Boolean)AccessController.doPrivileged(var10000::isEagerInit, this.getAccessControlContext());
            } else {
                isEagerInit = factory instanceof SmartFactoryBean && ((SmartFactoryBean)factory).isEagerInit();
            }

            if (isEagerInit) {
                this.getBean(beanName);
            }
        }
    }
```



#### beanFactory.getBean(beanName)以及doGetBean(beanName)

```java
public Object getBean(String name) throws BeansException {
  return this.doGetBean(name, (Class)null, (Object[])null, false);
}
```



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

populateBean(...)是DI部分，我放到另外一篇专门讲





