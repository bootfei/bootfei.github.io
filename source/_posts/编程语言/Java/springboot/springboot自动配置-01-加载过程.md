---
title: springboot-02-自动配置
date: 2020-12-03 12:12:59
tags:
---



**自动配置（SpringBoot最大特点）**

注解：2004年，jdk5发布，支持注解和组合注解; 2003年，Spring第一个项目开始；所以Spring 2.0开始支持注解，注解不是Spring boot的特点，反而是Spring特点。Spring支持xml 和注解两种方式。

### 自动加载总步骤

<font color="red">在Spring Boot中，内置类被Spring Boot自动加载的步骤以及实现</font>

- 首先项目有能力进行扫描，所以Application有@ComponentScan，对被注解的类所在的包进行扫描
- 然后只有被标记的类，才能在扫描中被Spring容器管理，所以有类有@Component，表明此类交给Spring容器管理
- 自动加载功能由@EnableAutoConfiguration标记，表明此类需要加载其他的相关类
  - @EnableAutoConfiguration表明此类需要加载哪些类呢？由Meta-Info/spring.factories中的key=EnableAutoConfiguration表明，value都是xxxxAutoConfiguration.class类
  - 类已经加载了，怎么创建对象呢？@Component表明该类需要创建对象

<font color="red">在Spring Boot中，自定义starter被Spring Boot自动加载的步骤以及实现</font>

- @SpringBootApplication包含了@EnalbeAutoConfiguration表明启动自动加载
- 自动加载会



### **@SpringBootApplication**

@SpringBootApplication 注解其实就是一个组合注解。包含有

- @Target
- @Retention
- @Documented
- @Inherited:表示注解会被子类自动继承。
- @SpringBootConfiguration 
- @ComponentScan
- @EnableAutoConfiguration

  

### **@SpringBootConfiguration**  = @Configuration

该注解与@Configuration 注解功能相同，其目的就是为 Spring Boot 专门创建的一个注解 

> @Configuration是Spring的注解

### **@ComponentScan** 

顾名思义，用于完成组件扫描。不过需要注意，其[仅仅用于配置组件扫描指令]()，并没有真正扫描，更没有装配其中的类，这个[真正扫描是由@EnableAutoConfiguration 完成的](https://www.hangge.com/blog/cache/detail_2807.html)。

看作者的注解：

> Either basePackageClasses or basePackages (or its alias value) may be specified to define specific packages to scan. If specific packages are not defined, scanning will occur [from the package of the class that declares this annotation]().

说明了Spring会扫描该类所在的包，这个包下的所有类及其子包中的类！！！

### 知识点补充 **@EnableXxx**

@EnableXxx 注解一般用于开启某一项功能，是为了简化代码的导入，即使用了该类注 解，就会自动导入某些类。所以该类注解是组合注解，一般都会包含一个@Import 注解，用 于导入指定的多个类，而被导入的类一般分为三种:[配置类、选择器与注册器](https://www.hangge.com/blog/cache/detail_2807.html)。

#### 配置类：

@Import 中指定的类一般以 Configuration 结尾，且该类上会注解@Configuration，表示当前类为配置类。

```java
  @Import(SchedulingConfiguration.class)
```

  

#### 选择器：

@Import 中指定的类一般以 Selector 结尾，且该类实现了 ImportSelector 接口，表示当前类会根据条件选择导入不同的类。

```java
  @Import(AutoConfigurationImportSelector.class)
```

  

#### 注册器：

@Import 指定的类一般以 Registrar 结尾，且该类实现了 ImportBeanDefinitionRegistrar接口，用于导入注册器，该类可以在代码运行时动态注册指定类的实例。

```java
  @Import(AspectJAutoProxyRegistrar.class)
```

  

### **@EnableAutoConfiguration**(属于@EnableXXX的[Selctor]())

该注解用于开启自动配置，是 Spring Boot 的核心注解，是一个组合注解。所谓自动配置是指，其会自动找到其所需要的starter以及用于自定义的类，然后交给 Spring 容器完成这些类的装配。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
```



@EnableAutoConfiguration扫描的自动配置类有2种：

- 一种是系统内置的类和starter的类，**由@Import(AutoConfigurationImportSelector.class)完成**
- 另外一种是Application内部自定义的,如程序员自己写的controller,service，**由@AutoConfigurationPackage完成**



##### **@Import**：完成系统内置类和starter类的扫描

```java
//AutoConfigurationImportSelector.class

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
      List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
      Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
      return configurations;
}
```



```java
//SpringFactoriesLoader.class
// loadFactoryNames() --> loadSpringFactories()  -->   "META-INF/spring.factories"

public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();
        return (List)loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}



private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            try {
                Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
                LinkedMultiValueMap result = new LinkedMultiValueMap();

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Entry<?, ?> entry = (Entry)var6.next();
                        List<String> factoryClassNames = Arrays.asList(StringUtils.commaDelimitedListToStringArray((String)entry.getValue()));
                        result.addAll((String)entry.getKey(), factoryClassNames);
                    }
                }

                cache.put(classLoader, result);
                return result;
            } catch (IOException var9) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var9);
            }
        }
    }
```



 **"META-INF/spring.factories"位置**

```xml
<artifactId>spring-boot-starter-web</artifactId>
	--->parent
<artifactId>spring-boot-starters</artifactId>
	--->parent
<artifactId>spring-boot-parent</artifactId>
	--->parent
<artifactId>spring-boot-dependencies</artifactId>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-autoconfigure</artifactId>
  <version>2.0.3.RELEASE</version>
</dependency>

```



![image-20220111132534741](/Users/qifei/Documents/blog/source/_posts/编程语言/Java/springboot/自动配置Spring-Factories.png)

从这里可以看到：系统内置的AutoConfiguration(starter)都被加载进来了，包括Redis, Aop等等

> 这些starter又引用了其他starter，这些starter同时也会被加载尽量





##### **@AutoConfigurationPackage**: 完成Applicaiton中用户自定义的类

用于导入[第二种类：用户自定义类](https://www.hangge.com/blog/cache/detail_2807.html)，即自动扫描包中的类。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import({Registrar.class})
public @interface AutoConfigurationPackage {
}
```



```java
//AutoConfigurationPackages.class

//Registrar是个内置类
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
  Registrar() {
  }

  //关键方法
  public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    AutoConfigurationPackages.register(registry, (new AutoConfigurationPackages.PackageImport(metadata)).getPackageName());
  }

  public Set<Object> determineImports(AnnotationMetadata metadata) {
    return Collections.singleton(new AutoConfigurationPackages.PackageImport(metadata));
  }
}
```



debug可以看到：扫描的是com.abc(用户的自定义的包)

![image-20220111133841003](/Users/qifei/Documents/blog/source/_posts/编程语言/Java/springboot/自动配置registerBeanDefinitions.png)





AutoConfigurationPackages.Registrar

- registerBeanDefinitions()
- register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0])); [//非常重要，可以看到第二个arg为项目的包名，即该类扫描了用户自定义的类]()





### 实践

#### Spring Boot整合Redis

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
    - RedisProperties类有@ConfigurationProperties(prefix="spring.redis")， 表明将配置文件中前缀为spring.redis的信息加载进来，并且将信息封装到RedisProperties.class中

#### MyBatis整合Spring Boot

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



## 自定义 **Starter**

### 创建starter

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

### 使用starter

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

  

## 常见问题问答

### starter和maven引入依赖有什么区别

- 相同点
  - 都引入了某个类的依赖
- 不同点
  - maven只是单纯的引入某个类的class文件，并不负责创建该类
  - starter不仅引入某个类的class文件，还负责交给Spring容器并创建对象
    - starter通过spring.factories通知spring容器，"我需要自动装配"
    - starter中定义了xxxService业务类，将核心业务交给spring容器创建对象
    - starter中封装了配置类，通知spring容器使用该配置初始化业务量对象

