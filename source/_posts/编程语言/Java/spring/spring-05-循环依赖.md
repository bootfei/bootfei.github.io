---
title: spring-04-事务和循环依赖
date: 2020-10-17 19:52:16
categories: [java,spring]
tags:
---

ref: 

1. https://mp.weixin.qq.com/s/gbKoBjq7BKEL70Jr1WNdtw
2. https://mp.weixin.qq.com/s/ZaGeEnOySH7OoZ-FXt_lvA



## 什么是循环依赖 

循环依赖其实就是循环引用，也就是[两个或者两个以上的bean互相持有对方作为一个field，最终形成闭环]()。

循环依赖分为：

（1）构造器的循环依赖（不可解决）：Spring中会拋出BeanCurrentlyInCreationException异常

（2）setter方法的循环依赖（可解决）：[在解决setter方法的循环依赖时]()，Spring采用的是[提前暴露对象的方法]()。

> 通过提前暴露原则，解决循环依赖：
>
> - 问题的本身来源于Spring对于暴露Bean对象和Java对于暴露对象的过程不同
>
>   - spring认为，一个Bean对象必须经历3步才能暴露，创建Bean的3个步骤：实例化new XXX()，依赖注入populateBean()，init-method初始化方法。
>   - java认为，一个对象在new以后就可以暴露。
>
> - 要解决这个问题，就得让Spring提前暴露对象，在new实例化的时候就得暴露
>

### constructor的循环依赖

[这个Spring解决不了，只能调整配置文件，将构造函数注入方式改为属性注入方式。]()

构造器循环依赖示例：

```java
@Service
public class ServiceA { 
	private ServiceB serviceB ; 
    
  //构造器循环依赖 
  @Resource
  public ServiceA(ServiceB serviceB) { this.serviceB = serviceB; } 
}

@Service
public class ServiceB { 
    private ServiceA serviceA ;
  
    @Resource
    public ServiceB(ServiceA serviceA) { this.serviceA = serviceA; } 
}
```



测试类：

```java
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'serviceA': Requested bean is currently in creation: Is there an unresolvable circular reference?
```



**Q: 为什么通过构造方法注入就无法解决？**

当JVM接收到一条new Object（）时候会执行下面的逻辑：

- 首先向内存中申请一块对象的内存区域
- 然后初始化对象成员变量信息并赋默认值（如 int类型为0，引用类型为nul）
- 最后执行对象的构造方法。构造方法执行完之前，这个对象的内存地址的都无法被获取。

当Spring使用构造方法注入时,

- Spring执行A对象的构造方法，但是构造方法属性依赖了B对象，必须先创建完B对象才能完成构造方法的执行
- Spring调用B对象的构造方法时，发现又依赖于A对象，所以B对象的构造方法也无法执行完毕，得先创建A
- Spring调用A对象的构造方法，....重复了第一个步骤

那么此时两个对象都没法申请到一个内存地址，当对象内存都生成不了试想，又如何为其依赖的属性赋值，显然不可能赋值为null。所以这种情况的循环依赖是没办法解决的，因为始终都没办法解决谁先创建的问题。



### setter方法循环依赖

```java
@Service
public class A {
    @Autowired
    private B b;
}
 
@Service
public class B {
    @Autowired
    private A a;
}
```



### Spring循环依赖发生的时机

Spring中的单例Bean创建的步骤为三步：

①：createBeanInstance：实例化，其实也就是 调用对象的构造方法实例化对象  ----> new ServiceA()

②：populateBean：填充属性，这一步主要是多bean的依赖属性进行填充

③：initializeBean：调用spring xml中的init() 方法。

从上面讲述的单例bean初始化步骤我们可以知道，如果是[构造器循环依赖]()，发生在第一步；如果是[field循环依赖]()，发生在第二步；

那么我们要解决循环引用也应该从初始化过程着手，对于单例来说，在Spring容器整个生命周期内，有且只有一个对象，所以很容易想到这个对象应该存在Cache中，[Spring为了解决单例的循环依赖问题，使用了二级缓存断开循环，三级缓存保证`AOP`代理的生命周期正常]()。 





## Spring检测循环依赖

[可以 Bean在创建的时候给其打个标记，如果递归调用回来发现正在创建中的话--->即可说明正在发生循环依赖。]()

DefaultSingletonBeanRegistry 

```java
private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16)); 
protected void beforeSingletonCreation(String beanName) { 
    if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) { 
        //抛出BeanCurrentlyInCreationException异常 
    } 
}
protected void afterSingletonCreation(String beanName) { 
    if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) { 
        //抛出IllegalStateException异常 
    } 
}
```

DefaultSingletonBeanRegistry#getSingleton 

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) { 
    synchronized (this.singletonObjects) { 
        Object singletonObject = this.singletonObjects.get(beanName); 
        if (singletonObject == null) {
            //... 
            // 创建之前，设置一个创建中的标识 , 非常重要！！！
            beforeSingletonCreation(beanName); 
            //... 
            try {
                // 调用匿名内部类获取单例对象 // 该步骤的完成，意味着bean的创建流程完成 
                singletonObject = singletonFactory.getObject();
                newSingleton = true; 
            }catch (IllegalStateException ex) { //... 
            }finally { 
                //... // 创建成功之后，删除创建中的标识
                afterSingletonCreation(beanName); 
            }
            // 将产生的单例Bean放入缓存中（总共三级缓存） 
            if (newSingleton) { 
                addSingleton(beanName, singletonObject);                          
            } 
        }
        return singletonObject; 
    } 
}
```

## Spring解决循环依赖问题

### 三级缓存解决该问题

DefaultSingletonBeanRegistry

```java
/** 第一级缓存 */ 
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256); 

/** 第二级缓存 */ 
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

/** 第三级缓存 */ 
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16); 
```

1. 一级缓存，singletonObjects，存储所有已创建完毕的单例 Bean （完整的 Bean）
2. 二级缓存，earlySingletonObjects，存储所有仅完成实例化，但还未进行属性注入和初始化的 Bean
3. 三级缓存，singletonFactories，存储能建立这个 Bean 的一个工厂，通过工厂能获取这个 Bean，延迟化 Bean 的生成，工厂生成的 Bean 会塞入二级缓存

这三个 map 是如何配合的呢？

1. 首先，获取单例 Bean 的时候会通过 BeanName 先去 singletonObjects（一级缓存） 查找完整的 Bean，如果找到则直接返回，否则进行步骤 2。
2. 看对应的 Bean 是否在创建中，如果不在直接返回找不到，如果是，则会去 earlySingletonObjects （二级缓存）查找 Bean，如果找到则返回，否则进行步骤 3
3. 去 singletonFactories （三级缓存）通过 BeanName 查找到对应的工厂，如果存着工厂则通过工厂创建 Bean ，并且放置到 earlySingletonObjects 中。
4. 如果三个缓存都没找到，则返回 null。

从上面的步骤我们可以得知，如果查询发现 Bean 还未创建，到第二步就直接返回 null，不会继续查二级和三级缓存。

返回 null 之后，说明这个 Bean 还未创建，这个时候会标记这个 Bean 正在创建中，然后再调用 createBean 来创建 Bean，而实际创建是调用方法 doCreateBean。

doCreateBean 这个方法就会执行上面我们说的三步骤：

1. 实例化

   - 在实例化 Bean 之后，会往第3级缓存 singletonFactories 塞入一个工厂，而调用这个工厂的 getObject 方法，就能得到这个 Bean。<!--要注意，此时 IOC 是不知道会不会有循环依赖发生的，但是它不管，反正往 singletonFactories 塞这个工厂，这里就是提前暴露。-->

2. 属性注入

   - 然后就开始执行属性注入，这个时候 A 发现需要注入 B，所以去 getBean(B)，此时又会走一遍上面描述的逻辑，到了 B 的属性注入这一步。

     此时 B 调用 getBean(A)，这时候第1级缓存singletonObjects里面找不到，于是去2级缓存earlySingletonObjects找，发现没找到，于是去三级缓存找，然后找到了。并且通过上面提前在三级缓存里暴露的工厂得到 A，然后将这个工厂从三级缓存里删除，并将 A 加入到二级缓存中。

3. 初始化

   - 紧接着 B 调用 initializeBean 初始化，最终返回，此时 B 已经被加到了一级缓存里 。

这时候就回到了 A 的属性注入，此时注入了 B，接着执行初始化，最后 A 也会被加到一级缓存里，且从二级缓存中删除 A。

Spring 解决依赖循环就是按照上面所述的逻辑来实现的。

重点就是在对象实例化之后，都会在三级缓存里加入一个工厂，提前对外暴露还未完整的 Bean，这样如果被循环依赖了，对方就可以利用这个工厂得到一个不完整的 Bean，破坏了循环的条件。

| A的createBean流程                                            | B的createBean流程                                            | 3个缓存map的变化                                             |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1. 实例化Instance A a = new A();                             |                                                              |                                                              |
| 一个持有a的ObjectFactory，被放入第三级缓存singletonFactories当中 |                                                              | singletonObjects: [] earlysingletonObjects: [] singletonObjectFactory: [a] |
| 2. 依赖注入populateBean()                                    |                                                              |                                                              |
| B b = getBean(b), 发现b的Bean不存在，则createBean            |                                                              |                                                              |
|                                                              | 1. 实例化Instance B b = new B()                              |                                                              |
|                                                              | 一个持有b的ObjectFactory，被放入第三级缓存singletonFactories当中 | singletonObjects: [] earlysingletonObjects: [] singletonObjectFactory: [a,b] |
|                                                              | 2. 依赖注入populateBean()                                    |                                                              |
|                                                              | A a = getBean(a)，发现a正在被创建                            |                                                              |
|                                                              | 从第1级缓存中没有找到，                                      |                                                              |
|                                                              | 从第2级缓存中没有找到，                                      |                                                              |
|                                                              | 从第三级缓存singletonObjectFactory获取a的ObjectFactory， EarlySingletonObject a = singletonObjectFactory.getObject() |                                                              |
|                                                              | 在3级缓存中清除a的singletonObjectFactory,  在2级缓存中添加a的earlySingletonObject | singletonObjects: [] earlysingletonObjects: [a] singletonObjectFactory: [b] |
|                                                              | b.setA(a)                                                    |                                                              |
|                                                              | 3. 初始化initMethod()                                        |                                                              |
|                                                              | B的bean创建完成,并且在1级缓存中添加b的singletonObject        | singletonObjects: [b] earlysingletonObjects: [a] singletonObjectFactory: [b] |
| 在1级缓存中找到b的bean                                       |                                                              | singletonObjects: [b] earlysingletonObjects: [a] singletonObjectFactory: [b] |
| a.setB(b)                                                    |                                                              |                                                              |
| 3. 初始化initMethod()                                        |                                                              |                                                              |
| A的bean创建完成,并且在1级缓存中添加a的singletonObject        |                                                              | singletonObjects: [a,b] earlysingletonObjects: [a] singletonObjectFactory: [b] |

### 解决循环依赖的代码 

AbstractAutowireCapableBeanFactory#doCreateBean 

```java
      // 解决循环依赖的关键步骤 
      boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&isSingletonCurrentlyInCreation(beanName)); //如果需要提前暴露单例Bean，则将该Bean放入三级缓存中
      if (earlySingletonExposure) { 
          //将刚创建的bean放入三级缓存中singleFactories(key是beanName，value是 FactoryBean) 
          addSingletonFactory(beanName,() -> getEarlyBeanReference(beanName, mbd, bean)); 
      }
      // Bean创建的第二步：依赖注入 
      // Bean创建的第三步：初始化Bean 
      // 是否提早暴露单例？上面已经计算过，循环依赖的时候，这个值就是true 
      if (earlySingletonExposure) { 
          // 如果是ClassA依赖ClassB，ClassB依赖ClassA 
          // 而且先实例化ClassA，在ClassA属性填充的时候，去实例化ClassB 
          // ClassB走到这里的时候，它的实例引用只是保存到三级缓存中。 
          // 但是getSingleton方法的第二个参数allowEarlyReference如果为false的话，就 是禁止使用三级缓存 
          // 【所以ClassB走到这里的时候，是从缓存中获取不到值的， earlySingletonReference为null。但是ClassB走到这里的时候，需要注入ClassA，说明ClassA已经从三级缓存里面取过 了，然后放入二级缓存了。ClassB走完了创建流程之后，ClassA也会走到这里，但是这个时候，ClassA的实例引 用是放到二级缓存中的。所以ClassA走到这里的时候，是从缓存中可以获取到值的，此处获取到的ClassA类型的earlySingletonReference，【其实是代理对象的引 用】。 
          Object earlySingletonReference = getSingleton(beanName, false); 
          if (earlySingletonReference != null) { 
              // 其实此处是将ObjectFactory产生的代理对象的实例引用，去替换ClassA正常流 程产生的原对象的引用。 
              if (exposedObject == bean) { exposedObject = earlySingletonReference; }
              // ...... 
          } 
      }
```



## Spring源码解决循环依赖

以`AbstractBeanFactory#doGetBean`为起点，该方法中有两个`getSingleton()`方法，

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    // 验证Bean的名字是否非法
    final String beanName = transformedBeanName(name);
    Object bean;
    
    // 从单例缓存中检查是否存在单例bean，第一次必定返回null
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }else {
        // Create bean instance.
        // 判断类是否是单例，prototype bean Spring是不会实例化的
        if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
                try {
                    // 创建bean的过程,对象的创建以及代理
                    return createBean(beanName, mbd, args);
                }
                catch (BeansException ex) {
              
                    // 从单例缓存中删除该Bean，因为其可能因为循环依赖提前放入到eagerly缓存中，所以删除所有依赖其的Bean
                    destroySingleton(beanName);
                    throw ex;
                }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }
    }
    return (T) bean;
}
```

第一次`getSingle(String beanName)`

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 单例池(一级缓存)，存放的是实例化好的bean
    Object singletonObject = this.singletonObjects.get(beanName);
    // 第一次必然是空，且singletonsCurrentlyInCreation不包含该beanName
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // 三级级缓存，存放临时对象，第一次必然为空
            singletonObject = this.earlySingletonObjects.get(beanName);
            // allowEarlyReference，在第一次调用getSingleton时写死为true,在第二次调用时,写死为false,禁用缓存工厂,返回提前暴露的AOP代理类
            if (singletonObject == null && allowEarlyReference) {
                // 二级缓存、存放的是工厂(用于代理)
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    // 将通过工厂创建的代理类，放入到三级缓存中
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 节省内存，没必要重新调用singletonFactory.getObject(),所以要remove
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

第二次调用`getSingleton(String beanName, ObjectFactory<?> singletonFactory)`，为重载后的方法。

```
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        // 从一级缓存中获取Bean
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            // 判断当前单例池是否正在被销毁
            if (this.singletonsCurrentlyInDestruction) {
                // throw new Exception
            }
            // 将当前bean放入到正在创建的集合singletonsCurrentlyInCreation中去
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
          	// ···
            try {
                // 实际创建Bean,getObject()方法实际调用的是AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
        	// ···
            finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                // Bean无论创建成功还是失败都从singletonsCurrentlyInCreation中移除
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                // 若创建成功则将其放入到一级缓存单例池中去，并从二级缓存(singleFactories)和三级缓存(earlySingleObjects)中移除
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```

上述源码中`singletonFactory.getObject()`实际调用的是`AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])`，具体如下:

```
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {
    RootBeanDefinition mbdToUse = mbd;
    // 从BeanDefinition Map中得到要创建的bean的类型
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }
  	// ···
    try {
        // 创建Bean的真正过程，
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isTraceEnabled()) {
            logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    } catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        // throw ex;
    } catch (Throwable ex) {
        // throw ex
    }
}
```

继续深入`doCreateBean(String, RootBeanDefinition, Object[])`方法

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {
    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 第二次调用后置处理器,创建对象
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 创建的实例对象
    final Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // 通过后置处理器合并BeanDefinition,拿到所有需要注入的属性
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            } catch (Throwable ex) {
               // throw ex
            }
            mbd.postProcessed = true;
        }
    }

    // 判断是否支持单例的循环依赖
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    // 若支持循环依赖则将bean提前暴露
    if (earlySingletonExposure) {
        // 当支持循环依赖时，就会提前暴露自己(为了其他对象可以注入自己)，第四次调用后置处理器
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

 	// ···· 省略Bean的注入过程和初始化过程(Aware回调、初始化方法等)

    if (earlySingletonExposure) {
        // 返回的是当前bean提前暴露的对象,为代理后的对象, allowEarlyReference=false,禁用singletonObjects缓存
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                // 将Bean替换成代理后的Bean
                exposedObject = earlySingletonReference;
            } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                                                               "Bean with name '" + beanName + "' has been injected into other beans [" +
                                                               StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                                                               "] in its raw version as part of a circular reference, but has eventually " +
                                                               "been" +
                                                               " " +
                                                               "wrapped. This means that said other beans do not use the final version of " +
                                                               "the" +
                                                               " " +
                                                               "bean. This is often the result of over-eager type matching - consider using" +
                                                               " " +
                                                               "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for " +
                                                               "example" +
                                                               ".");
                }
            }
        }
    }
    // ··· 实现disposableBean接口
    return exposedObject;
}
```

`addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory)`方法如下:

```
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            // 将singletonFactory放入到缓存工厂中,从earlySinglenObjects中移除
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
            // 已经注册的Bean Name集合
            this.registeredSingletons.add(beanName);
        }
    }
}
```

上述是循环依赖相关代码。以AB互相依赖为例，假设先加载A：

1. `getBean(a)`
2. 调用`getSingleton(a)`，第一次调用，因为此时Bean a未创建且3个缓存中都没有,所以返回null
3. 调用`getSingleton(a, ObjectFactory<?> singletonFactory)`，该方法中`singletonFactory.getObject()`实际调用的是`AbstractBeanFactory#createBean`
4. Bean a的实例化
5. 将Bean a提前暴露`addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean))`，将Bean a的对象工厂加入到`singletonFactories`中
6. Bean a的属性注入(`populateBean(beanName, mbd, instanceWrapper)`)，此时发现A依赖B，调用`getBean(b)`
7. 调用`getSingleton(b)`，第一次调用返回**null**
8. 第二次调用`getSingleton(ae, ObjectFactory<?> singletonFactory)`
9. Bean b的实例化
10. Bean b的属性注入，此时发现依赖A，调用`getBean(a)`，第一次执行`getSingleton(a)`，然后在`singletonFactories`中取得Bean a的工厂，调用`singletonFactory.getObject()`，这里会执行`addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean))`中的**lambda**表达式，实际是执行一个`BeanPostProcessor`后置处理器`AnnotationAwareAspectJAutoProxyCreator`，`AnnotationAwareAspectJAutoProxyCreator#getEarlyBeanReference`生成Bean a的代理类，即Bean b 注入Bean a的代理类。
11. Bean b完成属性注入、**Aware**回调、初始化后，`getSingleton(b, false)`调用第一个`getSingleton`并禁用`singletonFacotries`，从`earlySingleObjects`中获取提前暴露的对象，`exposedObject = earlySingletonReference`将Bean替换为`earlySingletonReference`并返回
12. Bean a完成属性注入、Aware回调、初始化
13. Bean a返回提前暴露的`earlySingleObject`。