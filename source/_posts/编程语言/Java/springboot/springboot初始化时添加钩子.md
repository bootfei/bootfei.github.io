---
title: springboot初始化时添加钩子
date: 2021-04-02 08:42:51
tags:
---

我们经常需要在容器启动的时候做一些钩子动作，比如注册消息消费者，监听配置等，今天就总结下`SpringBoot`留给开发者的7个启动扩展点。

## 容器刷新完成扩展点

### 监听容器刷新完成扩展点`ApplicationListener<ContextRefreshedEvent>`

#### 基本用法

熟悉`Spring`的同学一定知道，容器刷新成功意味着所有的`Bean`初始化已经完成，当容器刷新之后`Spring`将会调用容器内所有实现了`ApplicationListener<ContextRefreshedEvent>`的`Bean`的`onApplicationEvent`方法，应用程序可以以此达到监听容器初始化完成事件的目的。

```java
@Component
public class StartupApplicationListenerExample implements 
  ApplicationListener<ContextRefreshedEvent> {

    private static final Logger LOG 
      = Logger.getLogger(StartupApplicationListenerExample.class);

    public static int counter;

    @Override public void onApplicationEvent(ContextRefreshedEvent event) {
        LOG.info("Increment counter");
        counter++;
    }
}
```

#### 易错的点

这个扩展点用在`web`容器中的时候需要额外注意，在web 项目中（例如`spring mvc`），系统会存在两个容器，一个是`root application context`,另一个就是我们自己的`context`（作为`root application context`的子容器）。如果按照上面这种写法，就会造成`onApplicationEvent`方法被执行两次。解决此问题的方法如下：

```java
@Component
public class StartupApplicationListenerExample implements 
  ApplicationListener<ContextRefreshedEvent> {

    private static final Logger LOG 
      = Logger.getLogger(StartupApplicationListenerExample.class);

    public static int counter;

    @Override public void onApplicationEvent(ContextRefreshedEvent event) {
        if (event.getApplicationContext().getParent() == null) {
            // root application context 没有parent
            LOG.info("Increment counter");
            counter++;
        }
    }
}
```

#### 高阶玩法

当然这个扩展还可以有更高阶的玩法：**自定义事件**，可以借助`Spring`以最小成本实现一个观察者模式：

- 先自定义一个事件：

```
public class NotifyEvent extends ApplicationEvent {
    private String email;
    private String content;
    public NotifyEvent(Object source) {
        super(source);
    }
    public NotifyEvent(Object source, String email, String content) {
        super(source);
        this.email = email;
        this.content = content;
    }
    // 省略getter/setter方法
}
```

- 注册一个事件监听器

```
@Component
public class NotifyListener implements ApplicationListener<NotifyEvent> {

    @Override
    public void onApplicationEvent(NotifyEvent event) {
        System.out.println("邮件地址：" + event.getEmail());
        System.out.println("邮件内容：" + event.getContent());
    }
}
```

- 发布事件

```
@RunWith(SpringRunner.class)
@SpringBootTest
public class ListenerTest {
    @Autowired
    private WebApplicationContext webApplicationContext;

    @Test
    public void testListener() {
        NotifyEvent event = new NotifyEvent("object", "abc@qq.com", "This is the content");
        webApplicationContext.publishEvent(event);
    }
}
```

- 执行单元测试可以看到邮件的地址和内容都被打印出来了

### `SpringBoot`的`CommandLineRunner`接口

当容器上下文初始化完成之后，`SpringBoot`也会调用所有实现了`CommandLineRunner`接口的`run`方法，下面这段代码可起到和上文同样的作用：

```
@Component
public class CommandLineAppStartupRunner implements CommandLineRunner {
    private static final Logger LOG =
      LoggerFactory.getLogger(CommandLineAppStartupRunner.class);

    public static int counter;

    @Override
    public void run(String...args) throws Exception {
        LOG.info("Increment counter");
        counter++;
    }
}
```

对于这个扩展点的使用有额外两点需要注意：

- 多个实现了`CommandLineRunner`的`Bean`的执行顺序可以根据`Bean`上的`@Order`注解调整
- 其`run`方法可以接受从控制台输入的参数，跟`ApplicationListener<ContextRefreshedEvent>`这种扩展相比，更加灵活

```
// 从控制台输入参数示例
java -jar CommandLineAppStartupRunner.jar abc abcd
```

### `SpringBoot`的`ApplicationRunner`接口

这个扩展和`SpringBoot`的`CommandLineRunner`接口的扩展类似，只不过接受的参数是一个`ApplicationArguments`类，对控制台输入的参数提供了更好的封装，以`--`开头的被视为带选项的参数，否则是普通的参数

```
@Component
public class AppStartupRunner implements ApplicationRunner {
    private static final Logger LOG =
      LoggerFactory.getLogger(AppStartupRunner.class);

    public static int counter;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        LOG.info("Application started with option names : {}", 
          args.getOptionNames());
        LOG.info("Increment counter");
        counter++;
    }
}
```

比如：

```
java -jar CommandLineAppStartupRunner.jar abc abcd --autho=mark verbose
```

## `Bean`初始化完成扩展点

前面的内容总结了针对容器初始化的扩展点，在有些场景，比如监听消息的时候，我们希望`Bean`初始化完成之后立刻注册监听器，而不是等到整个容器刷新完成，`Spring`针对这种场景同样留足了扩展点：

### 1、`@PostConstruct`注解

`@PostConstruct`注解一般放在`Bean`的方法上，被`@PostConstruct`修饰的方法会在`Bean`初始化后马上调用：

```
@Component
public class PostConstructExampleBean {
    private static final Logger LOG 
      = Logger.getLogger(PostConstructExampleBean.class);

    @Autowired
    private Environment environment;

    @PostConstruct
    public void init() {
        LOG.info(Arrays.asList(environment.getDefaultProfiles()));
    }
}
```

### 2、 `InitializingBean`接口

`InitializingBean`的用法基本上与`@PostConstruct`一致，只不过相应的`Bean`需要实现`afterPropertiesSet`方法

```
@Component
public class InitializingBeanExampleBean implements InitializingBean {

    private static final Logger LOG 
      = Logger.getLogger(InitializingBeanExampleBean.class);

    @Autowired
    private Environment environment;

    @Override
    public void afterPropertiesSet() throws Exception {
        LOG.info(Arrays.asList(environment.getDefaultProfiles()));
    }
}
```

### 3、`@Bean`注解的初始化方法

通过`@Bean`注入`Bean`的时候可以指定初始化方法：

**`Bean`的定义**

```
public class InitMethodExampleBean {

    private static final Logger LOG = Logger.getLogger(InitMethodExampleBean.class);

    @Autowired
    private Environment environment;

    public void init() {
        LOG.info(Arrays.asList(environment.getDefaultProfiles()));
    }
}
```

**`Bean`注入**

```
@Bean(initMethod="init")
public InitMethodExampleBean initMethodExampleBean() {
    return new InitMethodExampleBean();
}
```

### 通过构造函数注入

`Spring`也支持通过构造函数注入，我们可以把搞事情的代码写在构造函数中，同样能达到目的

```
@Component 
public class LogicInConstructorExampleBean {

    private static final Logger LOG 
      = Logger.getLogger(LogicInConstructorExampleBean.class);

    private final Environment environment;

    @Autowired
    public LogicInConstructorExampleBean(Environment environment) {
        this.environment = environment;
        LOG.info(Arrays.asList(environment.getDefaultProfiles()));
    }
}
```

### 验证：`Bean`初始化完成扩展点执行顺序？

```java
@Component
@Scope(value = "prototype")
public class AllStrategiesExampleBean implements InitializingBean {

    private static final Logger LOG 
      = Logger.getLogger(AllStrategiesExampleBean.class);

    public AllStrategiesExampleBean() {
        LOG.info("Constructor");
    }

    //与implements InitializingBean联合使用
    @Override
    public void afterPropertiesSet() throws Exception {
        LOG.info("InitializingBean");
    }

    @PostConstruct
    public void postConstruct() {
        LOG.info("PostConstruct");
    }

    public void init() {
        LOG.info("init-method");
    }
}
```

实例化这个`Bean`后输出：

```
[main] INFO o.b.startup.AllStrategiesExampleBean - Constructor
[main] INFO o.b.startup.AllStrategiesExampleBean - PostConstruct
[main] INFO o.b.startup.AllStrategiesExampleBean - InitializingBean
[main] INFO o.b.startup.AllStrategiesExampleBean - init-method
```