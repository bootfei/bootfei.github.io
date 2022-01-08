---
title: springboot启动流程-01-启动流程
date: 2021-04-16 09:30:13
tags: [java,springboot,startup]
---



## SpringApplication#initialize()源码

```java
// SpringApplication.class

// 构造函数：在run()方法链路：2nd step中被调用
public SpringApplication(Object... sources) {
    initialize(sources);
}

// initailize()方法 在run()方法链路：2nd step 构造函数中被调用
private void initialize(Object[] sources) {
    if (sources != null && sources.length > 0) {
        this.sources.addAll(Arrays.asList(sources));
    }
    // 1st step: 推断应用类型是否是Web环境
    this.webEnvironment = deduceWebEnvironment();
    // 2nd step: 设置初始化器（Initializer）
    setInitializers((Collection) getSpringFactoriesInstances(
        ApplicationContextInitializer.class));
    // 3rd step: 设置监听器
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 4th step: 推断应用入口类
    this.mainApplicationClass = deduceMainApplicationClass();
}

//run()方法链路：1st step
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
  return run(new Class[]{primarySource}, args);
}

//run()方法链路：2nd step
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
  //我们可以发现其构造方法在这，其实调用了一个初始化的initialize方法
  return (new SpringApplication(primarySources)).run(args);
}

```

### 1st step:推断应用类型是否是Web环境

```java
//springApplication.class
// 相关常量
private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
        "org.springframework.web.context.ConfigurableWebApplicationContext" };

private boolean deduceWebEnvironment() {
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return false;
        }
    }
    return true;
}
```

这里通过判断classpath下是否存在 Servlet 和 ConfigurableWebApplicationContext 类来判断是否是Web环境， 所以在 Spring Boot 项目中 jar包 的引用不应该随意，不需要的依赖最好去掉。

### 2nd step:设置初始化器（Initializer）

```java
//2nd step:设置初始化器（Initializer）
setInitializers((Collection) getSpringFactoriesInstances(
            ApplicationContextInitializer.class));

//2.1
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

// 这里的入参type就是ApplicationContextInitializer.class
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // 2.2 加载所有ContextInitializer：使用Set保存names来去重 避免重复配置导致多次实例化
    Set<String> names = new LinkedHashSet<>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 2.3 实例化所有ContextInitializer：根据names来进行实例化
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    // 2.4 对实例进行排序 可用 Ordered接口 或 @Order注解 配置顺序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}

// 2.3
// 关键参数：
// type: org.springframework.context.ApplicationContextInitializer.class
// names: 上一步得到的names集合
private <T> List<T> createSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
        Set<String> names) {
    List<T> instances = new ArrayList<T>(names.size());
    for (String name : names) {
        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass
                    .getDeclaredConstructor(parameterTypes);
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException(
                    "Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}

```

这里出现了一个概念 - `上下文初始化器`。

**所谓的初始化器就是 org.springframework.context.ApplicationContextInitializer 的实现类，这个接口是这样定义的：**

```java
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

    /**
     * Initialize the given application context.
     * @param applicationContext the application to configure
     */
    void initialize(C applicationContext);

}
```

ApplicationContextInitializer是一个回调接口，它会在 ConfigurableApplicationContext 容器 refresh() 方法调用之前被调用，做一些容器的初始化工作。



getSpringFactoriesInstances() 方法会[加载]()所有配置的 `ApplicationContextInitializer` 并进行[实例化]()，加载 `ApplicationContextInitializer` 是在`SpringFactoriesLoader.loadFactoryNames` 方法里面进行的：

```java
//SpringFactoriesLoader.loadFactoryNames()方法
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
  String factoryClassName = factoryClass.getName();

  try {
      //获取类路径下的META-INF/spring.factories
      Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories"):ClassLoader.getSystemResources("META-INF/spring.factories");
      ArrayList result = new ArrayList();

      while(urls.hasMoreElements()) {
          URL url = (URL)urls.nextElement();
          Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
          String factoryClassNames = properties.getProperty(factoryClassName);
          result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
      }

      return result;
  } catch (IOException var8) {
      throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + "META-INF/spring.factories" + "]", var8);
  }
}
```

这个方法会尝试从类路径的 META-INF/spring.factories 读取相应配置文件，然后进行遍历，读取配置文件中Key为：org.springframework.context.ApplicationContextInitializer 的 value。以 spring-boot 这个包为例，它的 META-INF/spring.factories 部分定义如下所示：

```xml
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.context.embedded.ServerPortInfoApplicationContextInitializer
```

因此这4个类名会被读取出来，然后放入到集合中，准备实例化`createSpringFactoriesInstances()`操作。

初始化步骤很直观，类加载，确认被加载的类确实是org.springframework.context.ApplicationContextInitializer 的子类，然后就是得到构造器进行初始化，最后放入到实例列表中。

### 3rd step: 设置监听器（Listener）

实现方式与Initializer一样：

```java
//3rd step: 设置监听器（Listener）
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));

//3.1 这里的入参type是：org.springframework.context.ApplicationListener.class
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    //3.2 加载：同step2 
    Set<String> names = new LinkedHashSet<String>(
            SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    //3.3 实例化：同step2
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
            classLoader, args, names);
    //3.4
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

同样，还是以spring-boot这个包中的 spring.factories 为例，看看相应的 Key-Value ：

```xml
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener,\
org.springframework.boot.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.logging.LoggingApplicationListener
```

至于 ApplicationListener 接口，它是 Spring 框架中一个相当基础的接口，代码如下：

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

    /**
     * Handle an application event.
     * @param event the event to respond to
     */
    void onApplicationEvent(E event);

}
```

这个接口基于JDK中的 EventListener 接口，实现了观察者模式。对于 Spring 框架的观察者模式实现，它限定感兴趣的事件类型需要是 ApplicationEvent 类型的子类，而这个类同样是继承自JDK中的 EventObject 类。



### 4th step: 推断应用入口类（Main）

```java
//4th step: 推断应用入口类（Main）
this.mainApplicationClass = deduceMainApplicationClass();

private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```

它通过构造一个运行时异常，通过异常栈中方法名为main的栈帧来得到入口类的名字。

**获取堆栈信息的方式？**

```css
Thread.currentThread().getStackTrace()；
new RuntimeException().getStackTrace();
```

至此，对于SpringApplication实例的初始化过程就结束了。