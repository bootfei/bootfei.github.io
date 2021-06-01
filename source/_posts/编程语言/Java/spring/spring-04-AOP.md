---
title: spring-03-aop
date: 2020-10-17 19:52:16
categories: [java,spring]
tags:
---



# aop核心概念

## AOP相关术语介绍

- 和目标类有关
  - Joinpoint(连接点)：所谓连接点是指那些被拦截到的点。在spring中,这些点指的是方法,因为spring只支持方法类型的连接点 
  - Pointcut(切入点) ：所谓切入点是指我们要对哪些Joinpoint进行拦截的定义
  - Target(目标对象) ：代理的目标对象 
- 和增强类有关
  - Advice(通知/增强) ：所谓通知是指拦截到Joinpoint之后所要做的事情就是通知.通知分为前置通知,后置通知,异常通知,最 终通知,环绕通知(切面要完成的功能) 
  - Introduction(引介) （了解就行） ：引介是一种特殊的通知在不修改类代码的前提下, Introduction可以在运行期为类动态地添加一些方法或Field 
- 和两者都有关系
  - Aspect(切面) :是切入点和通知的结合，以后咱们自己来编写和配置的 



## AOP实现之Spring AOP

实现原理分析

- Spring AOP是通过[动态代理技术]()实现的
- 而动态代理是基于反射设计的。（关于反射的知识，请自行学习）
- 动态代理技术的实现方式有两种：基于接口的JDK动态代理和基于继承的CGLib动态代理。

### JDK动态代理

[目标对象必须实现接口]()

```java
// JDK代理对象工厂&代理对象方法调用处理器 
public class JDKProxyFactory implements InvocationHandler { 
    // 目标对象的引用
    private Object target; 
    // 通过构造方法将目标对象注入到代理对象中 
    public JDKProxyFactory(Object target) { 
        super(); 
        this.target = target; 
    }
    /*** @return */ 
    public Object getProxy() {
        // 如何生成一个代理类呢？ 
        // 1、编写源文件 
        // 2、编译源文件为class文件 
        // 3、将class文件加载到JVM中(ClassLoader) 
        // 4、将class文件对应的对象进行实例化（反射） 
        // Proxy是JDK中的API类 
        // 第一个参数：目标对象的类加载器 
        // 第二个参数：目标对象的接口 
        // 第二个参数：代理对象的执行处理器, 就是InvocationHandler
        Object object = Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this); 
        return object; 
    }
    /*** 代理对象会执行的方法 */ 
    @Override public Object invoke(Object proxy, Method method, Object[] args) throws Throwable { 
        Method method2 = target.getClass().getMethod("saveUser", null); 
        Method method3 = Class.forName("com.sun.proxy.$Proxy4").getMethod("saveUser", null); 
        System.out.println("目标对象的方法:" + method2.toString()); 
        System.out.println("目标接口的方法:" + method.toString()); 
        System.out.println("代理对象的方法:" + method3.toString()); System.out.println("这是jdk的代理方法"); 
        // 下面的代码，是反射中的API用法 
        // 该行代码，实际调用的是[目标对象]的方法 
        // 利用反射，调用[目标对象]的方法 O
        bject returnValue = method.invoke(target, args); 
        return returnValue; 
    } 
}
```

### Cglib动态代理

- 目标对象不需要实现接口
- 底层是通过继承目标对象产生代理子对象（代理子对象中继承了目标对象的方法，并可以对该方法进行增强）

```java
public class CgLibProxyFactory implements MethodInterceptor { 
    /*** @param clazz * @return */ 
    public Object getProxyByCgLib(Class clazz) { 
        // 创建增强器 Enhancer 
        enhancer = new Enhancer(); 
        // 设置需要增强的类的类对象 
        enhancer.setSuperclass(clazz); 
        // 设置回调函数 
        enhancer.setCallback(this); 
        // 获取增强之后的代理对象
        return enhancer.create(); 
    }
    /*** * Object proxy:这是代理对象，也就是[目标对象]的子类 * Method method:[目标对象]的方法 * Object[] arg:参数 * MethodProxy methodProxy：代理对象的方法 */ 
    @Override public Object intercept(Object proxy, Method method, Object[] arg, MethodProxy methodProxy) throws Throwable { 
        // 因为代理对象是目标对象的子类 
        // 该行代码，实际调用的是父类目标对象的方法 
        System.out.println("这是cglib的代理方法"); 
        // 通过调用子类[代理类]的invokeSuper方法，去实际调用[目标对象]的方法 
        Object returnValue = methodProxy.invokeSuper(proxy, arg); 
        // 代理对象调用代理对象的invokeSuper方法，而invokeSuper方法会去调用目标类的 invoke方法完成目标对象的调用 
        return returnValue; 
    } 
}
```



# aop使用

其实就是指的Spring + AspectJ整合，不过Spring已经将AspectJ收录到自身的框架中了，并且底层织入依然是采取的动态织入方式。 

## 添加依赖 

```xml
<!-- 基于AspectJ的aop依赖 --> 
<dependency> 
<groupId>org.springframework</groupId> 
<artifactId>spring-aspects</artifactId>
<version>5.0.7.RELEASE</version> 
</dependency> 

<dependency> 
<groupId>aopalliance</groupId> 
<artifactId>aopalliance</artifactId> 
<version>1.0</version> 
</dependency>
```

## 编写目标类和目标方法

编写接口和实现类（目标对象）: UserService接口 ,  UserServiceImpl实现类

配置目标类，将目标类交给spring IoC容器管理:

```xml
<context:component-scan base-package="sourcecode.ioc" />
```

## 使用XML实现 

编写通知（增强类，一个普通的类） 

```java
public class MyAdvice{
	public void log(){
		System.out.println("logging...");
	}
}
```

配置通知，将通知类交给spring IoC容器管理 

```xml
<bean name="MyAdvice" class="com.bootfei.MyAdvice"/>
```

配置AOP 切面

```xml
<aop:config>
	<aop:aspect ref="MyAdvice">
		<!--method表示指定增强类的功能, pointcut表示切入点，通过表达式表-->
	    <aop:before method="log" pointcut="execution(void com.bootfei.dao.UserDaoImpl.insert())"></aop:before>
    </aop:aspect>
</aop:config>
```

## 使用混合注解实现 

编写切面类（注意不是通知类，因为该类中可以指定切入点）

```java
@Component("myAspect")
@Aspect
public class MyAspect{
	@Before(value="execution(void com.bootfei.dao.UserDaoImpl.insert())")
	public void log(){
		System.out.println("logging...");
	}
    
  //环绕通知
  @Around(value="execution(* *.*(...))")
  public Object aroundAdvice(ProceedingJointPoint joinPoint){
    Object[] args= joinPoint.getArgs();
    Object rtValue = joinPoint.proceed(args);
  }
    
  //使用@PointCut注解在切面类中定义一个通用的切入点，其他通知可以引用该切入点
  @Before(value="MyAspect.fn()")
	public void log(){
		System.out.println("logging...");
	}
  
  @Before(value="execution(void com.bootfei.dao.UserDaoImpl.insert())")
  public void fun(){

  }
}
```

配置切面类，扫描成为Bean

```xml
<context:component-scan base-package="org.bootfei"/>
```

开启AOP自动代理

```xml
<aop:aspectj-autoproxy/>
```

## 使用纯注解实现 

```java
@Configuration
@ComponectScan(basePackages="org.bootfei")
@EnableAspectJAutoProxy
pubic class SpringConfiguartion{

}
```



# aop源码阅读

![在这里插入图片描述](https://img-blog.csdnimg.cn/20181204223004291.png)

## Spring AOP核心类解析

| 类名                                 | 作用概述                                                     |
| ------------------------------------ | ------------------------------------------------------------ |
| AopNamespaceHandler                  | AOP命名空间解析类。我们在用AOP的时候，会在Spring配置文件的beans标签中引入：xmlns:aop |
| AspectJAutoProxyBeanDefinitionParser | 解析<aop:aspectj-autoproxy />标签的类。在AopNamespaceHandler中创建的类。 |
| ConfigBeanDefinitionParser           | 解析<aop:config /> 标签的类。同样也是在AopNamespaceHandler中创建的类。 |
| AopNamespaceUtils                    | AOP命名空间解析工具类，在上面两个中被引用                    |



## step1:解析磁盘配置文件中AOP空间类型的标签

![](https://upload-images.jianshu.io/upload_images/838913-002aef5f6aeb7fd6.png)

> 根据自定义标签前面的值<aop:>，找NameSpaceHandler
>
> 然后再根据自定义标签后面的值<:config>，找BeanDefinitionParser

AopNamespaceHandlerSupport解析AOP空间类型的标签前面的标签

解析的目的都是为了将磁盘中的配置信息生成BeanDefinition，比如对于<aop: config>，解析以后内存中共有10个BeanDefinition。

> 解析aop:config标签，最终解析10个BeanDefinition。
>
> - 产生代理对象的类对应的BeanDefinition（一种）：[AspectJAwareAdvisorAutoProxyCreator]()
> - 通知类BeanDefinition（五种）：AspectJMethodBeforeAdvice、AspectJAfterAdvice、AspectJAfterReturningAdvice、AspectJAfterThrowingAdvice、AspectJAroundAdvice
> - 通知器BeanDefinition（一种）：DefaultBeanFactoryPointcutAdvisor、AspectJPointcutAdvisor
> - 切入点BeanDefinition（一种）：AspectJExpressionPointcut
> - 其他略，我也不懂

## step2:解析中配置文件中AOP空间

> 根据自定义标签，找到对应的BeanDefinitionParser，比如aop:config标签，就对应着ConfigBeanDefinitionParser。

[DefaultBeanDefinitionDocumentReader#parseBeanDefinitions 方法的第16行或者23行：]() 

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) { 
    // 加载的Document对象是否使用了Spring默认的XML命名空间（beans命名空间） 
    if (delegate.isDefaultNamespace(root)) { 
        // 获取Document对象根元素的所有子节点（bean标签、import标签、alias标签 和其他自定义标签context、aop等） 
        NodeList nl = root.getChildNodes(); 
        for (int i = 0; i < nl.getLength(); i++) { 
            Node node = nl.item(i); 
            if (node instanceof Element) { 
                Element ele = (Element) node; 
                // bean标签、import标签、alias标签，则使用默认解析规则 
                if (delegate.isDefaultNamespace(ele)) { 
                  parseDefaultElement(ele, delegate); 
                }
                //像context标签、aop标签、tx标签，则使用用户自定义的解析规则解析 元素节点 
                else {delegate.parseCustomElement(ele); } } } }
    else {
        // 如果不是默认的命名空间，则使用用户自定义的解析规则解析元素节点 
        delegate.parseCustomElement(root); 
    } 
}
```

解析完配置信息，最重要的是[生成BeanDefinition]()





找到NamespaceHandlerSupport类的 parse 方法第6行代码： 

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
    // NamespaceHandler里面初始化了大量的BeanDefinitionParser来分别处理不同的自 定义标签 
    // 从指定的NamespaceHandler中，匹配到指定的BeanDefinitionParser 
    BeanDefinitionParser parser = findParserForElement(element, parserContext); 
    // 调用指定自定义标签的解析器，完成具体解析工作 
    return (parser != null ? parser.parse(element, parserContext) : null);
}
```

## 产生AOP代理类流程分析

AspectJAwareAdvisorAutoProxyCreator的继承体系 

```
|-BeanPostProcessor 
	postProcessBeforeInitialization---初始化之前调用 
	postProcessAfterInitialization---初始化之后调用 

|--InstantiationAwareBeanPostProcessor 
	postProcessBeforeInstantiation---实例化之前调用 
	postProcessAfterInstantiation---实例化之后调用 
	postProcessPropertyValues---后置处理属性值 

|---SmartInstantiationAwareBeanPostProcessor 
	predictBeanType 
	determineCandidateConstructors 
	getEarlyBeanReference 

|----AbstractAutoProxyCreator 
	postProcessBeforeInitialization 
	postProcessAfterInitialization----AOP功能入口 		
	postProcessBeforeInstantiation 
	postProcessAfterInstantiation 
	postProcessPropertyValues ... 
	
|-----AbstractAdvisorAutoProxyCreator 
	getAdvicesAndAdvisorsForBean 
	findEligibleAdvisors 
	findCandidateAdvisors 
	findAdvisorsThatCanApply 
	
|------AspectJAwareAdvisorAutoProxyCreator 
	extendAdvisors 
	sortAdvisors
```



AbstractAutoProxyCreator类的 postProcessAfterInitialization 方法第6行代码： 

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException { 
    if (bean != null) { 
        Object cacheKey = getCacheKey(bean.getClass(), beanName); 
        if (!this.earlyProxyReferences.contains(cacheKey)) { // 使用动态代理技术，产生代理对象 
            return wrapIfNecessary(bean, beanName, cacheKey); 
        } 
    }
    return bean;
}
```

## 代理对象执行流程

主要去针对Jdk产生的动态代理对象进行分析，其实就是去分析InvocationHandler的invoke方法

入口：JdkDynamicAopProxy#invoke方法

