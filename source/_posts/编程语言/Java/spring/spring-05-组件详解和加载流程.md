---
title: spring-05-组件详解和加载流程
date: 2020-10-17 19:52:16
categories: [java,spring]
tags:
---

# Spring基本组成模块

（1）CoreContain模块：Core、bean、context、Expression Language。

（2）Data Access/integration（集成）模块：JDBC、ORM、OXM、JMS、Transaction.

（3）Web模块：WEB、Web-Servle、Web-Struts、Web-Portlet。

（4）AOP、Aspects、Instrumentation、Test.



# Spring核心组件详解

Spring核心组件只有Core、Context、Beans三个。core包侧重于帮助类，操作工具，beans包更侧重于bean实例的描述。context更侧重全局控制，功能衍生。

## Bean组件（IOC中存储的实际对象，由基础容器BeanFactory创建）

Bean组件主要解决：Bean 的定义、Bean 的创建以及对 Bean 的解析。

开发者关心Bean创建，其他由Spring内部帮你完成。

### Bean的创建

**（1）[Bean 的创建]()时典型的工厂模式，他的顶级接口是 BeanFactory，下图是这个工厂的继承层次关系：**

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2Fgn875ficGpAYiczicEvaHjOyMOOd1IP6sbfzhcyj0lfDJWh9GREeAhmlg/640)

BeanFactory 有三个子类：ListableBeanFactory、HierarchicalBeanFactory 和 AutowireCapableBeanFactory。但[最终的默认实现类是 DefaultListableBeanFactory]()。实现多接口是为了区分在 Spring 内部操作对象传递和转化时，对对象的数据访问所做的限制。 <!--这就是接口隔离原则-->

例如 [ListableBeanFactory 接口表示这些 Bean 是可列表的]()，[HierarchicalBeanFactory 表示的是这些 Bean 是有继承关系的]()，也就是每个 Bean 有可能有父 Bean。[AutowireCapableBeanFactory 接口定义 Bean 的自动装配规则]()。这四个接口共同定义了 Bean 的集合、Bean 之间的关系、以及 Bean 行为。

### Bean的定义

**（2）[Bean 的定义]()主要有 BeanDefinition 描述，如下图说明了这些类的层次关系：**

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2FT79cPBhHbVL49icicWiahCE0zJAGvfn2AKLXjAkUZtMGeBP9Eb6kicq2XQ/640)

Bean 的定义就是完整的描述了在 Spring 的配置文件中你定义的节点中所有的信息，包括各种子节点。当 Spring 成功解析你定义的一个节点后，在 Spring 的内部他就被转化成 BeanDefinition 对象。以后所有的操作都是对这个对象完成的。

### Bean的解析

**（3）[bean 的解析]()过程非常复杂，功能被分的很细，因为这里需要被扩展的地方很多，必须保证有足够的灵活性，以应对可能的变化。Bean 的解析主要就是对 Spring 配置文件的解析。这个解析过程主要通过下图中的类完成：**

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2FtAJ3fVegLzicgADJvM2ibYFJKkuoS0ehty9TI437OtBRCE5u0Mo4VOfw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## Content组件（IOC容器）

Context 在 Spring 的 org.springframework.context 包下，给 Spring 提供一个运行时的环境，用以保存各个对象的状态。

ApplicationContext 是 Context 的顶级父类，他除了能标识一个应用环境的基本信息外，他还继承了五个接口，这五个接口主要是扩展了 Context 的功能。下面是 Context 的类结构图：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2FYpKCQlcsSPw5FNfAvJjNL5j7s2GPGsa8kD8P7fWXQ6FUic7Y8H5t3dA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

[从上图中可以看出 ApplicationContext 继承了 BeanFactory，这也说明了 Spring 容器中运行的主体对象是 Bean]()，[另外 ApplicationContext 继承了 ResourceLoader 接口，使得 ApplicationContext 可以访问到任何外部资源]()，这将在 Core 中详细说明。<!--这也充分说明了，context是高级容器，因为继承了BeanFactory, 同时也能加载各种外部资源-->



ApplicationContext 的子类主要包含两个方面：

1. ConfigurableApplicationContext 表示该 Context 是可修改的，也就是在构建 Context 中用户可以动态添加或修改已有的配置信息，它下面又有多个子类，其中最经常使用的是可更新的 Context，即 AbstractRefreshableApplicationContext 类。
2. WebApplicationContext 顾名思义，就是为 web 准备的 Context 他可以直接访问到 ServletContext，通常情况下，这个接口使用的少。

再往下分就是按照构建 Context 的文件类型，接着就是访问 Context 的方式。这样一级一级构成了完整的 Context 等级层次。



总体来说 ApplicationContext 必须要完成以下几件事：

- 标识一个应用环境。
- 利用 BeanFactory 创建 Bean 对象。
- 保存对象关系表。
- 能够捕获各种事件。

Context 作为 Spring 的 Ioc 容器，基本上整合了 Spring 的大部分功能，或者说是大部分功能的基础。

## Core组件（定义了资源的访问方式）

Core 组件作为 Spring 的核心组件，他其中包含了很多的关键类，其中一个重要组成部分就是[定义了资源的访问方式]()。这种把所有资源都抽象成一个接口的方式很值得在以后的设计中拿来学习。

从下图可以看出 Resource 接口封装了各种可能的资源类型，也就是对使用者来说屏蔽了文件类型的不同。对资源的提供者来说，如何把资源包装起来交给其他人用这也是一个问题，我们看到：

<img src="https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2F6CukL8icI7ndtQDQoyphzeLBEbbKtSfBULe9fEsEgl7Nhz05QBA8EUA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片" style="zoom: 80%;" />

Resource 相关的类结构图

从上图可以看出 Resource 接口封装了各种可能的资源类型，也就是对使用者来说屏蔽了文件类型的不同。对资源的提供者来说，如何把资源包装起来交给其他人用这也是一个问题，我们看到：

- 屏蔽资源提供者：Resource 接口继承了 InputStreamSource 接口，这个接口中有个 getInputStream 方法，返回的是 InputStream 类。这样所有的资源都被可以通过 InputStream 这个类来获取，所以也屏蔽了资源的提供者。
- 资源加载：ResourceLoader 接口完成加载，他屏蔽了所有的资源加载者的差异，只需要实现这个接口就可以加载所有的资源，他的默认实现是 DefaultResourceLoader。

Resource与Context如何建立联系：

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2FCXZ99ns72lgF7LfvpJEqBcuDicnjeZPBd9vv8gPibx6sNWcHicgMbLLIg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上图可以看出，Context 是把资源的加载、解析和描述工作委托给了 ResourcePatternResolver 类来完成，他相当于一个接头人，他把资源的加载、解析和资源的定义整合在一起便于其他组件使用。Core 组件中还有很多类似的方式。

# Spring初始化逻辑（流程）

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2FA6e85dqzfoEYKvCib6Wf8PjpiaZQiaofXlhH4Mchyfju2EzIoxVUHicwng/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## 应用上下文加载类：

**(1)应用上下文（Application Context）**

负责装载bean的定义，并把它们组装起来，即装载配置文件。Spring应用上下文全权负责对象的组装。spring自带多种应用上下文的实现，它们之间的区别仅仅在于如何加载配置。

若Spring采用的是xml配置，则选择ClassPathXMLApplicationContext作为应用上下文比较合适。而对于基于java的配置，Spring提供了 AnnotationConfigApplicationContext.加载应用上下文：

**(2)Spring自带了多种类型的应用上下文。**

a、AnnotationConfigApplicationContext：从一个或多个基于Java配置类中加载Spring应用上下文。

b、AnnotationConfigWebApplicationContext：从一个或多个基于java的配置类中加载Spring Web应用上下文。

c、ClassPathXmlApplicationContext：从类路径下的一个或多个xml配置文件中加载上下文定义，把应用上下文定义文件作为类资源。

d、FileSystemXMLApplicationContext：从文件系统下的一个或多个xml配置文件中加载上下文定义。

e、xmlWebApplicationContext：从Web应用的一个或多个xml配置文件中加载上下文定义。

```java
//加载应用上下文的几种方式示例
//基于xml的配置
ClassPathXmlApplicationContext context=new ClassPathXmlApplicationContext(“classpath:spring.xml”);
//基于java的配置
AnnotaitionConfigApplicationContext context=new AnnotationConfigApplicationContext(“com.star.config.KnightConfig.class”); 
```

应用上下文准备就绪之后，我们就可以调用上下文的getBean()方法从Spring容器中获取bean。

```java
(MyUserDaoImpl)context.getBean("MyUserDaoImpl");
```

## BeanFactory 工厂创建

![图片](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqR6Jg1H8Gw5ryNDWeh5b2FjjcXWJGaIb5Im6hWI5zIiaF7kwP9atOWcd0lvibicdicgORc3bLXaoMbicg/640)

（1）这个方法就是构建整个 Ioc 容器过程的完整的代码，了解了里面的每一行代码基本上就了解大部分 Spring 的原理和功能了。

这段代码主要包含这样几个步骤：

- 构建 BeanFactory，以便于产生所需的“演员”。
- 注册可能感兴趣的事件。
- 创建 Bean 实例对象。
- 触发被监听的事件。

（2）refresh 也就是刷新配置，前面介绍了 Context 有可更新的子类，这里正是实现这个功能，当 BeanFactory 已存在是就更新，如果没有就新创建 <!--这一步非常重要，就是finishBeanFactoryInitialization(beanFactory)，高级容器Context与基础容器BeanFactory建立了联系-->

