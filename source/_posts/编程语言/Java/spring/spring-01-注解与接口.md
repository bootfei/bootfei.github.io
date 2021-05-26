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







## @Component，@Service等注解是如何被解析的？

#### @Component解析流程

##### 入口ContextNamespaceHandler#init()

Spring Framework2.0开始，引入可扩展的XML编程机制，该机制要求XML Schema命名空间需要与Handler建立映射关系。

该关系配置在相对于classpath下的spring-context-5.1.16.RELEASE.jar/META-INF/spring.handlers中。

![Image](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhpasNr82txibgDK8LDTzxQHic9hqyRl6FkJF7XCl2Qhu1P1RfiaFtd8SV5hMEib3Nu4zByr3UKkWvlsgQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

如上图所示 ContextNamespaceHandler对应xml配置文件的<context:... />,  分析的入口。

##### ComponentScanBeanDefinitionParser#parse()

ContextNamespaceHandler#init()方法注册了ComponentScanBeanDefinitionParser

![Image](https://mmbiz.qpic.cn/mmbiz_gif/JfTPiahTHJhpasNr82txibgDK8LDTzxQHicribzfxul4HV2iabRB8IxE6VPYEcZrDUdcMUUCOibNgRLf6RnejbpdVrtA/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)



其中有一个很重要的注释

> *// Actually scan for bean definitions and register them.*
>
> ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);

##### ClassPathBeanDefinitionScanner#doScan()

ComponentScanBeanDefinitionParser#parse()中有scanner.doScan()

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
      //findCandidateComponents 读Component配置文件,将Component配置文件变为内存中的BeanDefinition
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      for (BeanDefinition candidate : candidates) {
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
         if (candidate instanceof AbstractBeanDefinition) {
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         if (candidate instanceof AnnotatedBeanDefinition) {
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
         if (checkCandidate(beanName, candidate)) {
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            definitionHolder =
                  AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            beanDefinitions.add(definitionHolder);
            registerBeanDefinition(definitionHolder, this.registry);
         }
      }
   }
   return beanDefinitions;
}
```

上边的代码，从方法名，猜测：

findCandidateComponents：从classPath扫描组件，并转换为备选BeanDefinition，也就是要解析@Component。

##### ClassPathScanningCandidateComponentProvider#findCandidateComponents

findCandidateComponents在其父类ClassPathScanningCandidateComponentProvider 中。

```java
public class ClassPathScanningCandidateComponentProvider implements EnvironmentCapable, ResourceLoaderAware {
    //省略其他代码
    public Set<BeanDefinition> findCandidateComponents(String basePackage) {
       if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
          return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
       }
       else {
          return scanCandidateComponents(basePackage);
       }
    }

    private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
       Set<BeanDefinition> candidates = new LinkedHashSet<>();
       try {
          String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                resolveBasePackage(basePackage) + '/' + this.resourcePattern;
          Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
            //省略部分代码
          for (Resource resource : resources) {
            //省略部分代码
             if (resource.isReadable()) {
                try {
                   MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                   if (isCandidateComponent(metadataReader)) {
                      ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                      sbd.setSource(resource);
                      if (isCandidateComponent(sbd)) {
                         candidates.add(sbd);
                    //省略部分代码
          }
       }
       catch (IOException ex) {//省略部分代码 }
       return candidates;
    }
}
```

findCandidateComponents大体思路如下：

- `String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX resolveBasePackage(basePackage) + '/' + this.resourcePattern;` 将package转化为ClassLoader类资源搜索路径packageSearchPath，例如：`com.wl.spring.boot`转化为`classpath*:com/wl/spring/boot/**/*.class`
- `Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);` 加载搜素路径下的资源。
- `isCandidateComponent` 判断是否是备选组件
- `candidates.add(sbd);` 添加到返回结果的list

**ClassPathScanningCandidateComponentProvider#isCandidateComponent**

```java
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
    //省略部分代码
   for (TypeFilter tf : this.includeFilters) {
      if (tf.match(metadataReader, getMetadataReaderFactory())) {
         return isConditionMatch(metadataReader);
      }
   }
   return false;
}
```

includeFilters由registerDefaultFilters()设置初始值，有@Component，没有@Service啊？<!--别急，在Service解析流程有解释-->

```java
protected void registerDefaultFilters() {
   this.includeFilters.add(new AnnotationTypeFilter(Component.class)); //只有Component????没有Service!!!
   ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
   try {
      this.includeFilters.add(new AnnotationTypeFilter(
            ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
      
   }
   catch (ClassNotFoundException ex) {
      // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
   }
   try {
      this.includeFilters.add(new AnnotationTypeFilter(
            ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
     
   }
   catch (ClassNotFoundException ex) {
      // JSR-330 API not available - simply skip.
   }
}
```



##### ClassPathScanningCandidateComponentProvider#scanCandidateComponents

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
 //省略其他代码
 MetadataReader metadataReader   
             =getMetadataReaderFactory().getMetadataReader(resource);  
   if(isCandidateComponent(metadataReader)){
       //....
   }         
}

public final MetadataReaderFactory getMetadataReaderFactory() {
   if (this.metadataReaderFactory == null) {
      this.metadataReaderFactory = new CachingMetadataReaderFactory();
   }
   return this.metadataReaderFactory;
}
```

**确定metadataReader**

CachingMetadataReaderFactory继承自 SimpleMetadataReaderFactory，就是对SimpleMetadataReaderFactory加了一层缓存。

其内部的SimpleMetadataReaderFactory#getMetadataReader 为：

```java
public class SimpleMetadataReaderFactory implements MetadataReaderFactory{
    @Override
     public MetadataReader getMetadataReader(Resource resource) throws IOException {
         return new SimpleMetadataReader(resource, this.resourceLoader.getClassLoader());
    }
}
```

这里可以看出

```
MetadataReader metadataReader =new SimpleMetadataReader(...);
```

**查看match方法找重点方法**

<img src="https://mmbiz.qpic.cn/mmbiz_gif/JfTPiahTHJhpasNr82txibgDK8LDTzxQHic2Q7z0jlXTw7hz4rsU0fLicG4LOJJvPSiaqon6iajAVFaicGNk4BPjcL07A/640" alt="Image" style="zoom:50%;" />

AnnotationTypeFilter#matchself方法如下：

```
@Override
protected boolean matchSelf(MetadataReader metadataReader) {
   AnnotationMetadata metadata = metadataReader.getAnnotationMetadata();
   return metadata.hasAnnotation(this.annotationType.getName()) ||
         (this.considerMetaAnnotations && metadata.hasMetaAnnotation(this.annotationType.getName()));
}
```

是metadata.hasMetaAnnotation法，从名称看是处理元注解，我们重点关注



#### @Service解析流程

Spring如何处理@Service的注解的呢？？？？

查阅官方文档，下面这话：

https://docs.spring.io/spring/docs/5.0.17.RELEASE/spring-framework-reference/core.html#beans-meta-annotations

> @Component is a generic stereotype for any Spring-managed component. @Repository, @Service, and @Controller are specializations of @Component

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
// @Service 派生自@Component
@Component
public @interface Service {

   /**
    * The value may indicate a suggestion for a logical component name,
    * to be turned into a Spring bean in case of an autodetected component.
    * @return the suggested component name, if any (or empty String otherwise)
    */
   @AliasFor(annotation = Component.class)
   String value() default "";

}
```

@Component是@Service的元注解，Spring 大概率，在读取@Service，也读取了它的元注解，并将@Service作为@Component处理。



#### 逐步分析

##### 查找metadata.hasMetaAnnotation
metadata=metadataReader.getAnnotationMetadata();
metadataReader =new SimpleMetadataReader(...)
metadata= new SimpleMetadataReader#getAnnotationMetadata()

```java
//SimpleMetadataReader 的构造方法
SimpleMetadataReader(Resource resource, @Nullable ClassLoader classLoader) throws IOException {
   InputStream is = new BufferedInputStream(resource.getInputStream());
   ClassReader classReader;
   try {
      classReader = new ClassReader(is);
   }
   catch (IllegalArgumentException ex) {
      throw new NestedIOException("ASM ClassReader failed to parse class file - " +
            "probably due to a new Java class file version that isn't supported yet: " + resource, ex);
   }
   finally {
      is.close();
   }

   AnnotationMetadataReadingVisitor visitor =
            new AnnotationMetadataReadingVisitor(classLoader);
   classReader.accept(visitor, ClassReader.SKIP_DEBUG);

   this.annotationMetadata = visitor;
   // (since AnnotationMetadataReadingVisitor extends ClassMetadataReadingVisitor)
   this.classMetadata = visitor;
   this.resource = resource;
}
```

metadata=new SimpleMetadataReader(...).getAnnotationMetadata()= new AnnotationMetadataReadingVisitor（。。）

也就是说

```
metadata.hasMetaAnnotation=AnnotationMetadataReadingVisitor#hasMetaAnnotation
```

其方法如下：

```
public class AnnotationMetadataReadingVisitor{
    // 省略部分代码
@Override
public boolean hasMetaAnnotation(String metaAnnotationType) {
   Collection<Set<String>> allMetaTypes = this.metaAnnotationMap.values();
   for (Set<String> metaTypes : allMetaTypes) {
      if (metaTypes.contains(metaAnnotationType)) {
         return true;
      }
   }
   return false;
}
}
```

逻辑很简单，就是判断该注解的元注解在，在不在metaAnnotationMap中，如果在就返回true。

这里面核心就是metaAnnotationMap，搜索AnnotationMetadataReadingVisitor类，没有发现赋值的地方？？！。

推荐：[Java面试练题宝典](https://mp.weixin.qq.com/s?__biz=MzIyNDU2ODA4OQ==&mid=2247486846&idx=1&sn=75bd4dcdbaad73191a1e4dbf691c118a&scene=21#wechat_redirect)

##### 查找metaAnnotationMap赋值

回到SimpleMetadataReader 的方法，

```
//这个accept方法，很可疑，在赋值之前执行
SimpleMetadataReader(Resource resource, @Nullable ClassLoader classLoader) throws IOException {
//省略其他代码
AnnotationMetadataReadingVisitor visitor = new AnnotationMetadataReadingVisitor(classLoader);
classReader.accept(visitor, ClassReader.SKIP_DEBUG);
 this.annotationMetadata = visitor;
 }
```

发现一个可疑的语句：classReader.accept。

查看accept方法

```
public class ClassReader {
        //省略其他代码
public void accept(..省略代码){
    //省略其他代码
    readElementValues(
    classVisitor.visitAnnotation(annotationDescriptor, /* visible = */ true),
    currentAnnotationOffset,
     true,
    charBuffer);
}
}
```

查看readElementValues方法

```
public class ClassReader{
    //省略其他代码
private int readElementValues(
    final AnnotationVisitor annotationVisitor,
    final int annotationOffset,
    final boolean named,
    final char[] charBuffer) {
  int currentOffset = annotationOffset;
  // Read the num_element_value_pairs field (or num_values field for an array_value).
  int numElementValuePairs = readUnsignedShort(currentOffset);
  currentOffset += 2;
  if (named) {
    // Parse the element_value_pairs array.
    while (numElementValuePairs-- > 0) {
      String elementName = readUTF8(currentOffset, charBuffer);
      currentOffset =
          readElementValue(annotationVisitor, currentOffset + 2, elementName, charBuffer);
    }
  } else {
    // Parse the array_value array.
    while (numElementValuePairs-- > 0) {
      currentOffset =
          readElementValue(annotationVisitor, currentOffset, /* named = */ null, charBuffer);
    }
  }
  if (annotationVisitor != null) {
    annotationVisitor.visitEnd();
  }
  return currentOffset;
}
}
```

这里面的核心就是 annotationVisitor.visitEnd();

###### 确定annotationVisitor

这里的annotationVisitor=AnnotationMetadataReadingVisitor#visitAnnotation

源码如下，注意这里传递了metaAnnotationMap！！

```
public class AnnotationMetadataReadingVisitor{
@Override
public AnnotationVisitor visitAnnotation(String desc, boolean visible) {
   String className = Type.getType(desc).getClassName();
   this.annotationSet.add(className);
   return new AnnotationAttributesReadingVisitor(
         className, this.attributesMap,
              this.metaAnnotationMap, this.classLoader);
}
}
annotationVisitor=AnnotationAttributesReadingVisitor
```

###### 查阅annotationVisitor.visitEnd()

```
annotationVisitor=AnnotationAttributesReadingVisitor#visitEnd()
public class AnnotationAttributesReadingVisitor{
@Override
public void visitEnd() {
   super.visitEnd();

   Class<? extends Annotation> annotationClass = this.attributes.annotationType();
   if (annotationClass != null) {
      List<AnnotationAttributes> attributeList = this.attributesMap.get(this.annotationType);
      if (attributeList == null) {
         this.attributesMap.add(this.annotationType, this.attributes);
      }
      else {
         attributeList.add(0, this.attributes);
      }
      if (!AnnotationUtils.isInJavaLangAnnotationPackage(annotationClass.getName())) {
         try {
            Annotation[] metaAnnotations = annotationClass.getAnnotations();
            if (!ObjectUtils.isEmpty(metaAnnotations)) {
               Set<Annotation> visited = new LinkedHashSet<>();
               for (Annotation metaAnnotation : metaAnnotations) {
                  recursivelyCollectMetaAnnotations(visited, metaAnnotation);
               }
               if (!visited.isEmpty()) {
                  Set<String> metaAnnotationTypeNames = new LinkedHashSet<>(visited.size());
                  for (Annotation ann : visited) {
                     metaAnnotationTypeNames.add(ann.annotationType().getName());
                  }
                  this.metaAnnotationMap.put(annotationClass.getName(), metaAnnotationTypeNames);
               }
            }
         }
         catch (Throwable ex) {
            if (logger.isDebugEnabled()) {
               logger.debug("Failed to introspect meta-annotations on " + annotationClass + ": " + ex);
            }
         }
      }
   }
}
}
```

内部方法recursivelyCollectMetaAnnotations 递归的读取注解，与注解的元注解（读@Service，再读元注解@Component），并设置到metaAnnotationMap，也就是AnnotationMetadataReadingVisitor 中的metaAnnotationMap中。

#### 总结

大致如下：

```
ClassPathScanningCandidateComponentProvider#findCandidateComponents
```

**1.将package转化为ClassLoader类资源搜索路径packageSearchPath**

**2.加载搜素路径下的资源。**

**3.isCandidateComponent 判断是否是备选组件。**

内部调用的TypeFilter的match方法：

- `AnnotationTypeFilter#matchself中metadata.hasMetaAnnotation`处理元注解
- `metadata.hasMetaAnnotation=AnnotationMetadataReadingVisitor#hasMetaAnnotation`

就是判断当前注解的元注解在不在metaAnnotationMap中。

AnnotationAttributesReadingVisitor#visitEnd()内部方法recursivelyCollectMetaAnnotations 递归的读取注解，与注解的元注解（读@Service，再读元注解@Component），并设置到metaAnnotationMap

**4.添加到返回结果的list**