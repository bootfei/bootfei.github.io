---
title: spring-01-入门
date: 2020-10-17 19:52:16
categories: [java,spring]
tags:
---



## 接口

### CommandLineRunner接口

应用场景：基于目前业务需求需要提前将部分数据加载到Spring容器中。

解决方案：

- 定义静态常量，随着类的生命周期加载而提前加载（这种方式可能对于工作经验较少的伙伴，选择是最多的）

- 实现CommandLineRunner接口；容器启动之后，加载实现类的逻辑资源，已达到完成资源初始化的任务

- @PostConstruct；在具体Bean的实例化过程中执行，@PostConstruct注解的方法，会在构造方法之后执行

  加载顺序为：Constructor > @Autowired > @PostConstruct > 静态方法；

  特点：

  - 只有一个非静态方法能使用此注解
  - 被注解的方法不得有任何参数
  - 被注解的方法返回值必须为void
  - 被注解方法不得抛出已检查异常
  - 此方法只会被执行一次

- 实现InitializingBean接口；重写afterPropertiesSet()方法；



思路：SpringBoot提供的一种简单的实现方案，实现CommandLineRunner接口，实现功能的代码放在实现的run方法中加载，并且如果多个类需要夹加载顺序，则实现类上使用@Order注解，且value值越小则优先级越高。

#### 实践

基于CommandLineRunner接口建立两个实现类为RunnerLoadOne 、RunnerLoadTwo ；并设置加载顺序；

```java
@Component
@Order(1)
public class RunnerLoadOne implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        ClassDo classDo = SpringContextUtil.getBean(ClassDo.class);
        classDo.setClassName("Java");
        System.out.println("------------容器初始化bean之后,加载资源结束-----------");
    }
}

@Component
@Order(2)
public class RunnerLoadTwo implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        ClassDo bean = SpringContextUtil.getBean(ClassDo.class);
        System.out.println("依赖预先加载的资源数据：" + bean.getClassName());
    }
}
```

启动主实现类，看到console打印的结果如下：

```
o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8666 (http) with context path ''
2020-08-06 21:20:14.693  INFO 6612 --- [           main] com.qxy.InformalEssayApplication         : Started InformalEssayApplication in 121.651 seconds (JVM running for 173.476)
------------容器初始化bean之后,加载资源结束-----------
依赖预先加载的资源数据：Java
```

#### 验证

底层如何实现的：主启动类debugger

##### run()方法

跟进run方法后，一路F6直达以下方法

```java
public ConfigurableApplicationContext run(String... args) {
   StopWatch stopWatch = new StopWatch();
   //设置线程启动计时器
   stopWatch.start();
   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
   //配置系统属性：默认缺失外部显示屏等允许启动
   configureHeadlessProperty();
   //获取并启动事件监听器，如果项目中没有其他监听器，则默认只有EventPublishingRunListener
   SpringApplicationRunListeners listeners = getRunListeners(args);
   //将事件广播给listeners
   listeners.starting();
   try {
       //对于实现ApplicationRunner接口，用户设置ApplicationArguments参数进行封装
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
      //配置运行环境：例如激活应用***.yml配置文件      
      ConfigurableEnvironment environment = prepareEnvironment(listeners,
            applicationArguments);
      configureIgnoreBeanInfo(environment);
      //加载配置的banner(gif,txt...)，即控制台图样
      Banner printedBanner = printBanner(environment);
      //创建上下文对象，并实例化
      context = createApplicationContext();
      exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
      //配置SPring容器      
      prepareContext(context, environment, listeners, applicationArguments,
            printedBanner);
      //刷新Spring上下文，创建bean过程中      
      refreshContext(context);
      //空方法，子类实现
      afterRefresh(context, applicationArguments);
      //停止计时器：计算线程启动共用时间
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass)
               .logStarted(getApplicationLog(), stopWatch);
      }
      //停止事件监听器
      listeners.started(context);
      //开始加载资源
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, listeners, exceptionReporters, ex);
      throw new IllegalStateException(ex);
   }
   listeners.running(context);
   return context;
}
```

本篇文章主要是熟悉SpringBoot的CommandLineRunner接口实现原理。直接进入正题CallRunners()方法内部。

##### callRunners方法

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    //将实现ApplicationRunner和CommandLineRunner接口的类，存储到集合中
   List<Object> runners = new ArrayList<>();
   runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
   runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
   //按照加载先后顺序排序
   AnnotationAwareOrderComparator.sort(runners);
   for (Object runner : new LinkedHashSet<>(runners)) {
      if (runner instanceof ApplicationRunner) {
         callRunner((ApplicationRunner) runner, args);
      }
      if (runner instanceof CommandLineRunner) {
         callRunner((CommandLineRunner) runner, args);
      }
   }
}
```

##### callRunner方法

```java
private void callRunner(CommandLineRunner runner, ApplicationArguments args) {
   try {
       //调用各个实现类中的逻辑实现
      (runner).run(args.getSourceArgs());
   }
   catch (Exception ex) {
      throw new IllegalStateException("Failed to execute CommandLineRunner", ex);
   }
}
```

到此结束，再跟进run()方法，就可以看到我们实现的资源加载逻辑啦~

## 注解

### @Autowire

#### 实践

##### 注解在field上

```
@Autowire
xxxService service;
```

##### 注解在Collection上

**Spring 会自动将 xxxEntity接口的实现类注入到这个Map中。前提是你这个实现类得是交给Spring 容器管理的。**

这个Map的key值就是你的 bean id，你可以用@Component("value")的方式设置，若干用默认的方式的话，就是首字母小写。value值则为对应的策略实现类。

```
@Autowrei
Map<String, xxxEntity> map;

-----
@Component
public xxxEntity1 extends xxxEntity1{...}

@Component
public xxxEntity2 extends xxxEntity1{...}
```



##### 注解在方法上



### @Qualifier

#### 官方的介绍

> This annotation may be used on a field or parameter as a qualifier for candidate beans when autowiring. It may also be used to annotate other custom annotations that can then in turn be used as qualifiers.

简单的理解就是：

- 在使用@Autowire自动注入的时候，加上@Qualifier(“test”)可以指定注入哪个对象；
- 可以作为筛选的限定符，我们在做自定义注解时可以在其定义上增加@Qualifier，用来筛选需要的对象。

#### 实践

##### 与@Autowire联合使用

@Autowired和@Qualifier这两个注解的连用在这个位置就类似 @Resource(name=“testClass1”)

> 定义了两个TestClass对象，分别是testClass1和testClass2。如果在另外一个对象中直接使用@Autowire去注入的话，spring肯定不知道使用哪个对象，那么会抛出异常 required a single bean, but 2 were found

```java
@Configuration
public class TestConfiguration {
   @Bean("testClass1")
   TestClass testClass1(){
       return new TestClass("TestClass1");
   }
   @Bean("testClass2")
   TestClass testClass2(){
       return new TestClass("TestClass2");
   }
}
```

下面是正常的引用

```java
@RestController
public class TestController {
    //此时这两个注解的连用就类似 @Resource(name="testClass1")
    @Autowired
    @Qualifier("testClass1")
    private TestClass testClass;

    @GetMapping("/test")
    public Object test(){
        return testClassList;
    }

}
```



##### 作为筛选的限定符

```java
@Configuration
public class TestConfiguration {
    //我们调整下在testClass1上增加@Qualifier注解
    @Qualifier
    @Bean("testClass1")
    TestClass testClass1(){
        return new TestClass("TestClass1");
    }

    @Bean("testClass2")
    TestClass testClass2(){
        return new TestClass("TestClass2");
    }
}

@RestController
public class TestController {
    //我们这里使用一个list去接收testClass的对象
    @Autowired
    List<TestClass> testClassList= Collections.emptyList();
    
    @GetMapping("/test")
    public Object test(){
        return testClassList;
    }
}
```

我们调用得到的结果是

```
[
     {
        "name": "TestClass1"
     },
    {
       "name": "TestClass2"
    }
]
```

我们可以看到所有的testclass都获取到了。

接下来我们修改下代码

```
@RestController
public class TestController {

    @Qualifier //我们在这增加注解
    @Autowired
    List<TestClass> testClassList= Collections.emptyList();

    @GetMapping("/test")
    public Object test(){
        return testClassList;
    }
}
```

和上面代码对比就是在接收参数上增加了@Qualifier注解，结果如下：

```
[
     {
        "name": "TestClass1"
     }
]
```

返回结果只剩下增加了@Qualifier注解的TestClass对象。另外，@Qualifier注解是可以指定value的，这样我们可以通过values来分类筛选想要的对象了，