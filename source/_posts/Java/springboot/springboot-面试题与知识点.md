---
title: springboot-面试题与知识点
date: 2021-02-17 11:13:15
tags:
---

# 对于Spring Boot来说，最为重要的注解应该就是@SpringBootApplication了，请谈 一下你对这个注解的认识。

 Spring Boot中的@SpringBootApplication是一个组合注解。除了基本的元注解外，其 还组合了三个很重要的注解:

- @SpringBootConfiguration:其等价于@Configuration，表示当前类为一个配置类。
- @ComponentScan:该注解用于配置应用中Bean的扫描指令，但其并没有进行真正的扫描
- @EnableAutoConfiguration:这是核心注解，其开启了自动配置。对内置自动配置类的加载及对自定义 Bean 的扫描都是由该注解完成的。

# Spring Boot 中有一个注解@ComponentScan，请谈一下你对这个注解的认识。 

@ComponentScan是@SpringBootApplication注解的一个组成注 解，用于配置使用@Configuration 定义的组件的扫描指令，并且支持同时使用 Spring 配置文件中的<context:component-scan/>标签。该注解的注释说，该注解仅仅就 是配置了一下用于进行组件扫描的指令参数，并没有进行扫描，真正扫描并装配这些类是 @EnableAutoConfiguration 完成的。而这些指令参数就是通过该注解的属性进行配置的，例 如扫描哪里，扫描之前可以先进行怎样的过滤等。如果这些注解属性都没有配置，则默认会 扫描当前注解所标注类所在的包及其子孙包。

不过，对于这点，Spring Boot1.x 版本中仅会扫描当前标注类所在包的子孙包，不会扫 描标注类所在的包。但Spring Boot2.x版本中默认会扫描当前注解所标注类所在的包及其子 孙包。

对于一个 Spring Boot 工程，与自动配置相关的 Bean 均是由 Spring 容器管理的。而这些 Bean 的类型根据创建者的不同可以分为两种:

- 一种是由程序员自定义的组件类，例如我们 自己定义的处理器类、Service 类等;
- 另一种是框架本身已经定义好的自动配置相关类。

自定义类由@ComponentScan 指定的扫描指令进行扫描，而框架自身的自动配置相关类 在一个配置文件中存放，将来会被加载到内存。这两种类型的类都会由注解 @EnableAutoConfiguration 交给 Spring 容器来管理。

# Spring Boot 有@EnableAutoConfiguration，对这个注解的认识。

首先，@Enable 开头这一类注解一般用于开启某一项功能，是为了简化代码的导入，即使用了该类注解，就会自动导入某些类。所以该类注解是组合注解，一般都会组合一个 @Import 注解，用于导入指定的多个类。@EnableXxx 的功能主要体现在这些被导入的类上， 而被导入的类一般有三种:

- 配置类
- 可以实现动态选择的选择器
- 可以完成动态注解的注册器

然后，Spring Boot 中的注解@ EnableAutoConfiguration 是@SpringBootApplication 注解的一个组成注解，该注解用于完成自动配置相关的自定义类及内置类的加载。其本身也是一个 组合注解。除了元注解外，还组合了@Import 与@AutoConfigurationPackage 两个注解。具体 分工如下:

- @Import: 用于加载SpringBoot中内置的及导入starter中META-INF/spring.factory配置中的自动配置类。

- @AutoConfigurationPackage: 用于扫描、加载并注册自定义的组件类。

# META-INF/spring.factory 配置文件对于 Spring Boot 的自动配置很重要，为什么? 

无论是 Spring Boot 内置的 META-INF/spring.factory 配置文件，还是导入 Starter 依赖中 的 META-INF/spring.factory 配置文件，对于 Spring Boot 自动配置来说很重要是因为，其包含 了一个 key-value 对，key 为 EnableAutoConfiguration 的全限定性类名，而 value 则为当前应 用中所有可用的自动配置类。所有可用的自动配置类都在这里声明，所以该文件对于 Spring Boot 自动配置来说很重要。

#  Spring Boot 官方给出的 Starter 工程的命名规范吗具体是什么? 

Spring Boot 官方给出的 Starter 工程的命名需要遵循如下规范:

- Spring 官方定义的Starter格式为:spring-boot-starter-{name}，如spring-boot-starter-web。 
- 非官方Starter命名格式为:{name}-spring-boot-starter，如dubbo-spring-boot-starter。

# @Configuration 所注解的类称为配置类，其中的@Bean 方法可以创建相应实例，@Bean 方法实例的创建顺序什么?

 @Configuration 所注解的类中的@Bean 方法实例的创建顺序即这些@Bean 方法的执 行顺序。首先可以为这些方法的执行添加执行条件，例如使用以@ConditionOn 开头的条件 注解。在这些条件都满足的情况下，这些方法的执行顺序即为其在@Configuration 注解类中 的定义顺序，先定义者先执行，其对应实例先创建。

# 你曾定义过 Starter 吗?简单说一下定义的大体步骤。

对于自定义 Starter，其工程命名格式为{name}-spring-boot-starterConfiguration。工程 需要导入配置处理器依赖。然工程中需要定义如下的类与配置:

- 定义核心业务类，这是该Starter存在的意义。

- 定义自动配置类，其完成对核心业务类的实例化，并交给Spring容器。

- 若核心业务类中需要从配置文件获取配置数据，还需要定义一个用于封装配置文件中相关属性的类。

- 定义META-INF/spring.factories配置文件，用于对自动配置类进行注册。

# 在自动配置类中一般会涉及到很多的条件注解，简单介绍几个你了解的条件注解。

在自动配置类定义中的确会用于到很多的条件注解，条件注解一般都是以 @ConditionalOn 开头的，例如:

  - @ConditionalOnClass():条件判断注解，其可以标注在类与方法上，表示当参数指定的类在类路径下存在时才会创建当前类的 Bean，或当前方法指定的 Bean。

  - @ConditionalOnMissionBean:条件判断注解，其可以标注在类与方法上，表示当容器中不存在当前类或方法类型的对象时，将去创建一个相应的 Bean。

  - @ConditionalOnBean():条件判断注解，其可以标注在类与方法上，表示当在容器中存在指定类的实例时才会创建当前类的 Bean，或当前方法指定的 Bean。


# 谈一下你对 Spring Boot 启动过程的了解。

 Spring Boot 的启动从大的方面来说需要经过以下两大阶段:加载 Spring Boot 配置文 件 application.yml，与完成自动配置。

而自动配置又包含以下两个大的步骤:加载自动配置相关的类，与扫描并注册自定义的 组件类。

加载Spring Boot配置文件是在启动类的run()方法执行时加载的。其大体要经历运行环 境准备、为环境准备过程添加监听、发布环境准备事件等步骤。

自动配置则是由组合注解 @SpringBootApplication 所包含的子注解 @EnableAutoConfiguration 完成。其不仅加载了 META-INF/spring.factories 中内置的自动配置 相关类，还完成了自定义类的加载与注册。若存在第三方 Starter，则其会将该 Starter 中 META-INF/spring.factories 中的自动配置相关类加载并装配。

# 请简单谈一下你对 Spring Boot 自动配置的理解。

Spring Boot与SSM传统开发相比，其最大的特点是自动配置，不用再在xml文件中 做大量的配置了。自动配置的实现主要是通过自动配置类来完成的，自动配置类存在于两类 位置:一个是 Spring Boot 框架中内置的，一是从外部引入的 Starter 中。

具体来说，自动配置类的作用就是根据条件创建核心业务类的实例到 Spring 容器中， 以备该 Starter 的引用工程类中注入。当然，自动配置类还有一个作用:若创建核心业务类 时需要获取配置文件中的相关属性值，其也会将这些属性值封装为一个属性实例，以备核心 业务类使用。当然，自动配置类需要在 META-INF/spring.factories 中注册。

所以自动配置其实就是能够自动获取配置文件中的属性并创建相应核心业务类实例。



# 现在准备解析 Spring Cloud 中某子框架的源码，那么从哪里开始解析? 

对于一个 未曾阅读过的子框架源码，我认为从自动配置类开始解析可能是一个不错的选择。

我们知道 [Spring Cloud 是通过 Spring Boot 将其它第三方框架集成进来的]()。[Spring Boot 最大的特点就是自动配置]()，我们可以[通过导入相关 Starter 来实现需求功能的自动配置、相关核心业务类实例的创建等]()。也就是说，核心业务类都是集中在自动配置类中的。所以从这里 下手分析应该是个不错的选择。

那么从哪里可以找到这个自动配置类呢?从导入的 starter 依赖工程的 META-INF 目录中 的 spring.factory 文件中可以找到。该文件的内容为 key-value 对，查找 EnableAutoConfiguration 的全限定性类名作为 key 的 value，这个 value 就是我们要找到的自动配置类。

# @EnableConfigurationProperties 注解对于 Starter 的定义很重要

@EnableConfigurationProperties 注解在 Starter 定义时主要用于读取 application.yml 配 置文件中相关的属性，并封装到指定类型的实例中，以备 Starter 中的核心业务实例使用。

具体来说，它就是开启了对@ConfigurationProperties 注解的 Bean 的自动注册，注解到 Spring 容器中。这种 Bean 有两种注册方式:在配置类使用@Bean 方法注册，或直接使用该 注解的 value 属性进行注册。若在配置类中使用@Bean 注册，则需要在配置类中定义一个 @Bean 方法，该方法的返回值为“使用@ConfigurationProperties 注解标注”的类。若直接 使用该注解的 value 属性进行注册，则需要将这个“使用@ConfigurationProperties 注解标注” 的类作为 value 属性值出现即可。

# Spring Boot 中定义了很多条件注解，这些注解一般用于对配置类的控制。在这些条件注解中有一个@ConditionalOnMissingBean 注解,请谈一下你对它的认识。 

@ConditionalOnMissingBean 注解是 Spring Boot 提供的众多条件注册中的一个。其表示的意义是，当容器中没有指定名称或指定类型的 Bean 时，该条件为 true。不过，这里需 要强调一点的是，这里要查找的“容器”是可以指定的。通过 search 属性指定。其 search 的范围有三种:仅搜索当前配置类容器;搜索所有层次的父类容器，但不包含当前配置类容 器;搜索当前配置类容器及其所有层次的父类容器，这个是默认搜索范围。