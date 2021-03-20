---
title: springboot-02-自动配置
date: 2020-12-03 12:12:59
tags:
---

# Spring boot特点

## 自动配置（最大特点）

## 注解

2004年，jdk5发布，支持注解和组合注解; 2003年，Spring第一个项目开始；所以Spring 2.0开始支持注解，注解不是Spring boot的特点，反而是Spring特点。Spring支持xml 和注解两种方式。

# 自动加载总步骤

<font color="red">在Spring Boot中，内置类被Spring Boot自动加载的步骤以及实现</font>

- 首先项目有能力进行扫描，所以Application有@ComponentScan，对类所在的包进行扫描
- 然后只有被标记的类，才能在扫描中被Spring容器管理，所以有类有@Component，表明此类交给Spring容器管理
- 自动加载功能由@EnableAutoConfiguration标记，表明此类需要加载其他的相关类
  - @EnableAutoConfiguration表明此类需要加载哪些类呢？由Meta-Info/spring.factories中的key=EnableAutoConfiguration表明，value都是xxxxAutoConfiguration.class类
  - 类已经加载了，怎么创建对象呢？@Component表明该类需要创建对象

<font color="red">在Spring Boot中，自定义starter被Spring Boot自动加载的步骤以及实现</font>

- @SpringBootApplication包含了@EnalbeAutoConfiguration表明启动自动加载
- 自动加载会

# 自动配置源码解析

## 解析**@SpringBootApplication**

@SpringBootApplication 注解其实就是一个组合注解。包含有

- @Target
- @Retention
- @Documented
- @Inherited:表示注解会被子类自动继承。
- @SpringBootConfiguration 
- @ComponentScan
- @EnableAutoConfiguration

  

### 元注解

前四个是专门(即只能)用于对注解进行注解的，称为元注解。

- @Target
- @Retention
- @Documented
- @Inherited:表示注解会被子类自动继承。

### **@SpringBootConfiguration** 

查看该注解的源码注解可知，该注解与@Configuration 注解功能相同，仅表示当前类为一个 JavaConfig 类，其就是为 Spring Boot 专门创建的一个注解。表明该类交给Spring容器管理。

### **@ComponentScan** 

顾名思义，用于完成组件扫描。不过需要注意，其[仅仅用于配置组件扫描指令]()，并没有真正扫描，更没有装配其中的类，这个[真正扫描是由@EnableAutoConfiguration 完成的](https://www.hangge.com/blog/cache/detail_2807.html)。

看作者的注解：

> Either basePackageClasses or basePackages (or its alias value) may be specified to define specific packages to scan. If specific packages are not defined, scanning will occur [from the package of the class that declares this annotation]().

说明了Spring会扫描该类所在的包，这个包下的所有类及其子包中的类！！！

### **@EnableXxx**

@EnableXxx 注解一般用于开启某一项功能，是为了简化代码的导入，即使用了该类注 解，就会自动导入某些类。所以该类注解是组合注解，一般都会包含一个@Import 注解，用 于导入指定的多个类，而被导入的类一般分为三种:[配置类、选择器与注册器](https://www.hangge.com/blog/cache/detail_2807.html)。

- 配置类

  - @Import 中指定的类一般以 Configuration 结尾，且该类上会注解@Configuration，表示当前类为配置类。

  - ```java
    @Import(SchedulingConfiguration.class)
    ```

    

- 选择器

  - @Import 中指定的类一般以 Selector 结尾，且该类实现了 ImportSelector 接口，表示当前类会根据条件选择导入不同的类。

  - ```java
    @Import(AutoConfigurationImportSelector.class)
    ```

    

- 注册器

  - @Import 指定的类一般以 Registrar 结尾，且该类实现了 ImportBeanDefinitionRegistrar接口，用于导入注册器，该类可以在代码运行时动态注册指定类的实例。

  - ```java
    @Import(AspectJAutoProxyRegistrar.class)
    ```

    

## 解析**@EnableAutoConfiguration**

该注解用于开启自动配置，是 Spring Boot 的核心注解，是一个组合注解。所谓自动配置是指，其会自动找到其所需要的starter以及用于自定义的类，然后交给 Spring 容器完成这些类的装配。

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
```

> 它扫描的相关类有2种：一种是系统内部定义好的,如starter, 另外一种是自定义的,如controller,service

["META-INFO/spring.factories"非常重要的文件, 里面有个key = EnableAutoConfiguratin, value是众多自动配置类，如Redis配置类, Mongo配置类]()

### **@Import**

- 用于导入[第一种类：框架本身所包含的自动配置相关的类](https://www.hangge.com/blog/cache/detail_2807.html)。其参数 AutoConfigurationImportSelector 类，该类用于导入自动配置的类。

- [只是加载类的名称（我觉得更准确的说，是封装类的名称），并没有创建实例]()，什么时候创建呢？

- ```java
  @Import(AutoConfigurationImportSelector.class)
  ```

  - [具体加载框架本身的自动配置类的过程如下]()

  - AutoConfigurationImportSelector.class：

    - getCandidateConfigurations()
    - SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());

  - SpringFactoriesLoader.class

    - loadFactoryNames

    - loadSpringFactories

      - ```java
        Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
        ```

      - 非常重要！最终@Import实现了加载"[META-INF/spring.factories]()"这个文件。那么这个文件在哪里呢？

        - 项目的source目录下没有，继续
        - Pom.xml中有spring-boot-starter-web, 查看内部依赖，发现有spring-boot-starter,继续查看内部依赖，有spring-boot-autoconfigure依赖，在其项目下source目录，找到了！！！

### **@AutoConfigurationPackage** 

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
```

- 用于导入[第二种类：用户自定义类](https://www.hangge.com/blog/cache/detail_2807.html)，即自动扫描包中的类。

- ```java
  @Import(AutoConfigurationPackages.Registrar.class)
  ```

  - AutoConfigurationPackages.Registrar
    - registerBeanDefinitions()
    - register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0])); [//非常重要，可以看到第二个arg为项目的包名，即该类扫描了用户自定义的类]()

# **application.yml** 的加载

application.yml 文件对于 Spring Boot 来说是核心配置文件，至关重要，那么，该文件是 如何加载到内存的呢?需要从启动类的 run()方法开始跟踪。

- xxxApplication: 
  - SpringApplication.run(DemoApplication.class, args);
- SpringApplication:  
  - (new SpringApplication(primarySources)).run(args);
  - this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
  - listeners.environmentPrepared(bootstrapContext, (ConfigurableEnvironment)environment);
- SpringApplicationRunListeners
  - listener.environmentPrepared(environment);
- EventPublishingRunListener
  - initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
- SimpleApplicationEventMulticaster
  - this.invokeListener(listener, event);
  - this.doInvokeListener(listener, event);

- .....

[自己跟一下就行]()

# 实战

## 实战1:加载框架类 Spring Boot与Redis整合

> SpringBoot整合了Redis

> 总步骤：
>
> - 首先@SpringBootApplication的@Import注解，通过spring-boot-starter下的spring.factories, 加载了框架自定义的RedisAutoConfiguration类
> - 然后RedisAutoConfiguration类加载了配置文件中的"spring.redis"

- [在 spring.factories 中有一个 RedisAutoConfiguration 类，通过前面的分析我们知道，该类 一定会被 Spring 容器自动装配](https://www.hangge.com/blog/cache/detail_2807.html)。但自动装配了就可以读取到 Spring Boot 配置文件中 Redis 相关的配置信息了?这个类与 Spring Boot 配置文件是怎么建立的联系?
- RedisAutoConfiguration 类
  - 有@ConditionOn(RedisOperations.class),注解
    - 表明如果路径下有RedisOperation.class才会生效; 
    - RedisOperations.class就是Redis的操作对象
  - 有@EnableConfigurationProperties(RedisProperties.class)注解
    - RedisProperties类有@ConfigurationProperties(prefix="spring.redis")<!--录播课有讲-->, 表明将配置文件中前缀为spring.redis的信息加载进来，并且将信息封装到RedisProperties.class中

## 实战2:加载自定义类 MyBatis与Spring Boot整合

> Mybatis整合了Spring Boot

> 总步骤：
>
> - 首先@SpringBootApplication的@注解，通过spring-boot-starter下的spring.factories, 加载了框架自定义的RedisAutoConfiguration类
> - 然后RedisAutoConfiguration类加载了配置文件中的"spring.redis"

- 在 External Libraries 中找到 mybatis-spring-boot-starter 依赖。
  - 但是该依赖下却没有META-INF/spring.factories文件。再往下找依赖。
- 而该依赖又依赖于 mybatis-spring-boot-autoconfigure。
  - 其 META-INF 中有 spring.factories 文件，打开这个文件 我们找到了 Mybatis 的自动配置类，即MyBatisAutoConfiguration.class类
- MyBatisAutoConfiguration.class类
  - 有@ConditionOnClass注解
  - 有@ConditionOnBean注解
  - 有@EnableConfigurationProperties注解
  - 有@AutoConfigurationAfter(DataSourceAutoConfiguration.class)
    - 表明DataSourceAutoConfiguration.class配置完以后，才能配置MyBatisAutoConfiguration.class



# 自定义 **Starter**

## 创建starter

> 总步骤
>
> - 导入spring-boot-configuration-processor依赖
> - 创建封装类，使用@ConfigurationProperties("xxxx.xxxx")获取配置文件中的信息并封装
> - 创建自定义配置类
>   - @Configuration必须使用，因为要交给Spring容器管理
>   - @ConditionalOnClass(WrapService.class)必须使用，因为starter的意义就是使用其中的服务
>   - @EnableConfigurationProperties(WrapProperties.class)必须使用，因为要获取封装好的配置信息
>   - 创建@Bean的业务示例对象, return new XXXService();
> - 创建Spring.factories, 用来通知Spring该starter的自动装配所需要的配置类在哪里。

前面的代码中，无论是 Spring Boot 中使用 Web、Test，还是 MyBatis、Dubbo，[都是通过导入一个相应的 Starter 依赖，然后由 Spring Boot 自动配置完成的](https://www.hangge.com/blog/cache/detail_2807.html)。那么，如果我们自己 的某项功能也想通过自动配置的方式应用到 Spring Boot 中，为 Spring Boot 项目提供相应支持，需要怎样实现呢?同样，我们需要定义自己的 Starter。

- 命名: Starter 工程的命名需要遵循的规范
  - Spring官方定义的Starter格式为:spring-boot-starter-{name} 
  - 非官方定义的Starter格式为:{name}-spring-boot-starter

- 需求

  - 下面我们自定义一个我们自己的 Starter，实现的功能是:为用户提供的字符串添加前辍 与后辍，而前辍与后辍定义在 yml 或 properties 配置文件中。
  - 例如，用户输入的字符串为 China， application.yml 配置文件中配置的前辍为$$$，后辍为+++，则最终生成的字符串为 $$$China+++。

- 实现

  - 导入依赖

    - 配置处理器依赖
  
    - ```xml
      <dependency>
        <groupId>org.springframework.boot</groupId> 
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    	</dependency>
      ```
  
  - 定义配置属性封装类
  
    - 我们指定当前类用于封装来自于 Spring Boot 核心配置文件中的以 some.service 开头的 beore 与 after 属性值。
  
    - ```java
      import lombok.Data;
      import org.springframework.boot.context.properties.ConfigurationProperties;
      /**
       * 封装配置文件中的如下属性：
       * wrap.service.prefix
       * wrap.service.suffix
       */
      @ConfigurationProperties("wrap.service")
      @Data
      public class WrapProperties {
          private String prefix;
          private String suffix;
      }
      ```
  
  - 定义自动配置类
  
    - 为了加深大家对“自动配置类与配置文件属性关系”的理解，这里再增加一个功能:为 some.service 再增加一个组装开关，一个 boolean 属性 enable，当 enable 属性值为 true 时， 或没有设置 some.service.enable 属性时才进行组装，若 enable 为 false，则不进行组装。
  
    - ```java
      @Configuration
      @ConditionalOnClass(WrapService.class)
      @EnableConfigurationProperties(WrapProperties.class)
      public class WrapAutoConfiguration {
          @Autowired
          private WrapProperties properties;
      
          // 注意，以下两个@Bean方法的先后顺序不能颠倒，否则会创建2个WrapService Bean。可以通过 两个都加上@ConditionalOnMissingBean，保证只有1个WrapService Bean
          @Bean
          @ConditionalOnProperty(name = "wrap.service.enable", havingValue = "true", matchIfMissing = true)
          public WrapService wrapService() {
              return new WrapService(properties.getPrefix(), properties.getSuffix());
          }
      
          @Bean
          @ConditionalOnMissingBean
          public WrapService wrapService2() {
              return new WrapService("", "");
          }
      }
      ```
  
  - 创建 **spring.factories** 文件
  
    - 在 resources/META-INF 目录下创建一个名为 spring.factories 的文件。该配置文件是一个 键值对文件，键是固定的，为 EnableAutoConfiguration 类的全限定性类名，而值则为我们自定义的自动配置类。
  
    - ```
      org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.abc.config.WrapAutoConfiguration
      ```
  
      



## 使用starter

> 总步骤
>
> - 导入自定义的starter依赖，所以必须保证install到maven库中
> - 修改配置文件
> - 修改controller

- 导入依赖

  - ```xml
    <dependency>
      <groupId>com.abc</groupId>
      <artifactId>wrap-spring-boot-starter</artifactId>
      <version>0.0.1-SNAPSHOT</version>
    </dependency>
    ```

    

- 修改配置文件

  - ```yaml
    wrap:
      service:
        enable: true
        prefix: AAA-
        suffix: -BBB
    ```

- 修改controller

  - 使用自定义starter中的业务类WrapService

  - ```java
    @RestController
    public class WrapController {
        @Autowired
        private WrapService service;
    
        @RequestMapping("/wrap/{param}")
        public String wrapHandler(@PathVariable("param") String word) {
            return service.wrap(word);
        }
    
    }
    ```

  

# 常见问题问答

## starter和maven引入依赖有什么区别

- 相同点
  - 都引入了某个类的依赖
- 不同点
  - maven只是单纯的引入某个类的class文件，并不负责创建该类
  - starter不仅引入某个类的class文件，还负责交给Spring容器并创建对象
    - starter通过spring.factories通知spring容器，"我需要自动装配"
    - starter中定义了xxxService业务类，将核心业务交给spring容器创建对象
    - starter中封装了配置类，通知spring容器使用该配置初始化业务量对象

