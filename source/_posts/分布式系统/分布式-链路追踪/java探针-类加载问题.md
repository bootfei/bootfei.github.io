---
title: java探针-类加载问题
date: 2021-05-22 11:08:25
tags:
---

得益于Java SE 6提供的Instrumentation接口。基于Instrumentation可开发运行时修改class字节码的Java Agent应用（Java探针），可在[类加载之前替换类的字节码]()、或在[类加载之后通过重新加载类方式修改类的字节码]()。

只是实现运行时修改class字节码还不足以称为“探针”。基于Instrumentation开发的Java Agent，只需要在Java应用启动命令上加上虚拟机参数“-javaagent”指定Java Agent应用jar包的位置，而不需要在工程项目中引入其jar包，即可将探针插入应用代码的各个角落。通过[与应用使用不同的类加载实现环境隔离]()，让人有种Java Agent是吸附在应用上运行的错觉。



## NoClassDefFoundError

### 原因

由父类加载器加载的类，不能引用子类加载器加载的类，否则会抛出NoClassDefFoundError。

JDK提供的`java.*`类都由启动类加载器加载。如果我们在`java agent`中修改`java.*`包下的类，插入调用logback打印日记的代码，结果会怎样？由于`java agent`包下的logback由AppClassLoader加载，而加载`java.*`包下的类是BootClassLoader(AppClassLoader的父类加载器)，在`java.*`包下的类中插入调用logback打印日记的代码，首先在加载`java.*`包下的类时，jvm会查看BootClassLoader有没有加载过这个类，如果没有加载过尝试加载，但BootClassLoader加载不了logback包的类，而启动类加载器不会向子类加载器去询问，即使子类加载器加载了这个类。所以就会出现NoClassDefFoundError。

### 解决

如果非要修改java包下的类，且非要在java包下的类中访问项目中我们编写的类或者第三方jar包提供的类、或者我们编写的javaagent包下的类，如何避免NoClassDefFoundError呢？

研究下Arthas:

- 用于接收埋点代码上报事件的类（Spy）：

```java
public final class Spy {

    public static Method beforMethod;
    public static Method completeMethod;

    public static void before(String className, String methodName, String descriptor, Object[] params) {
        if (beforMethod != null) {
            try {
                beforMethod.invoke(null, className, methodName, descriptor, params);
            } catch (IllegalAccessException | InvocationTargetException e) {
            }
        }
    }

    public static void complete(Object returnValueOrThrowable, String className, String methodName, String descriptor) {
        if (completeMethod != null) {
            try {
                completeMethod.invoke(null, returnValueOrThrowable, className, methodName, descriptor);
            } catch (IllegalAccessException | InvocationTargetException e) {
            }
        }
    }
}
```

> before：方法执行之前上报；
>
> complete：方法return之前或者抛出异常之前上报，当方法抛出异常时，第一个参数为异常，否则第一个参数为返回值
>
> methodName: 上报方法

- 将Spy放在一个独立的jar包下，在premain、agentmain方法中调用Instrumentation的appendToBootstrapClassLoaderSearch方法，将Spy类所在的jar包交由启动类加载器扫描加载，如下代码所示。

```
// agent-spy.jar
String agentSpyJar = jarPath[1];
File spyJarFile = new File(agentSpyJar);
instrumentation.appendToBootstrapClassLoaderSearch(new JarFile(spyJarFile));
```

### 验证

在Spy类中打印类加载器，如果打印的结果为null，则说明Spy类是由启动类加载器加载的。

```
public final class Spy {
    static {
        System.out.println("Spy class loader is " + Spy.class.getClassLoader());
    }
    //.......
}
```

## 实现Agent与应用环境隔离

隔离是避免Agent污染应用自身，使开发Java Agent无需考虑引入的jar包是否与目标应用引入的jar包冲突。

### 与springboot隔离失败原因

Java Agent与Spring Boot应用相遇时会发生什么？

Spring Boot应用打包后，将Agent附着到应用启动可能会抛出醒目的NoClassDefFoundError异常，背后的原因是[Agent与打包后的Spring Boot应用使用了不同的类加载器]()。

- Agent的jar包由AppClassLoader加载。我们可能会在Agent中调用被监控的SpringBoot应用的代码，也可能调用Agent依赖的第三方jar包的API，而这些jar包恰好在SpringBoot应用中也有导入，就可能会出现NoClassDefFoundError。
- SpringBoot应用打包后，JVM进程启动入口不再是我们写的main方法，而是SpringBoot生成的启动类。SpringBoot使用自定义的类加载器（LaunchedClassLoader）加载jar中的类和第三方jar包中的类，该类加载器的父类加载器为AppClassLoader。也就是说，SpringBoot应用打包后，加载java agent包下的类加载器是SpringBoot的类加载器的父类。

> 疑惑？这在IDEA中测试是不会发生的
>
> 因为在IDEA中，项目的class文件和第三方库是通过AppClassLoader加载的，而使用-javaagent指定的jar也是通过AppClassLoader加载，所以在idea中测试不会遇到这个问题。

### 解决

让加载agent包不使用AppClassLoader加载器加载，而是使用自定义的类加载器加载。

参考Alibaba开源的Arthas的实现，自定义URLClassLoader加载agent包以及agent依赖的第三方jar包。

由于premain或者agentmain方法所在的类由jvm使用AppClassLoader所加载，所以必须将agent拆分为两个jar包。核心功能放在agent-core包下，premain或者agentmain方法所在的类放在agent-boot包下。在premain或者agentmain方法中使用自定义的URLClassLoader类加载器加载agent-core。

- 自定义类加载器OnionClassLoader，继承URLClassLoader，如下代码所示：

```java
public class OnionClassLoader extends URLClassLoader {

    public OnionClassLoader(URL[] urls) {
        super(urls, ClassLoader.getSystemClassLoader().getParent());
    }

    @Override
    protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        final Class<?> loadedClass = findLoadedClass(name);
        if (loadedClass != null) {
            return loadedClass;
        }
        // 优先从parent（SystemClassLoader）里加载系统类，避免抛出ClassNotFoundException
        if (name != null && (name.startsWith("sun.") || name.startsWith("java."))) {
            return super.loadClass(name, resolve);
        }
        try {
            Class<?> aClass = findClass(name);
            if (resolve) {
                resolveClass(aClass);
            }
            return aClass;
        } catch (Exception e) {
            // ignore
        }
        return super.loadClass(name, resolve);
    }

}
```

同时在构造方法中指定OnionClassLoader的父类加载器为AppClassLoader的父类加载器。

`ClassLoader.getSystemClassLoader()`：获取系统类加载器（AppClassLoader）



- 在premain或者agentmain方法中使用OnionClassLoader类加载器加载agent-core。

```java
File agentJarFile = new File(agentJar);
final ClassLoader agentLoader = new OnionClassLoader(new URL[]{agentJarFile.toURI().toURL()});

Class<?> transFormer = agentLoader.loadClass("com.msyc.agent.core.OnionClassFileTransformer");

Constructor<?> constructor = transFormer.getConstructor(String.class);
Object instance = constructor.newInstance(opsParams);

instrumentation.addTransformer((ClassFileTransformer) instance);
```

> - 根据agent-core.jar所在绝对路径构造OnionClassLoader；
> - 加载agent-core.jar下的ClassFileTransformer；
> - 使用反射创建ClassFileTransformer实例；
> - 将ClassFileTransformer添加到Instrumentation;



OnionClassFileTransformer类所依赖的agent-core包下的类，自然也会被使用OnionClassLoader类加载器加载，包括agent-core依赖的第三方jar包。

![图片](https://mmbiz.qpic.cn/mmbiz_png/yibbONzdtFf2t8JVXHNXRpibzSgFAqUAMkfyP8gypoicPR23f3cqb0icdKtGjjAicYowOAHJr07HqlV9YPbqqqPE4qg/640)

## **适配webmvc框架**

生成分布式调用链日记的难点在于方法埋点和方法调用日记串连。

分布式调用链日记串连的方式有多种，笔者采用的是最简单的方式：打点id+打点时间。

- 对于同进程内的同线程，可用打点id将调用的方法串连起来，根据打点时间与一个累加器的值排序方法调用日记。
- 对于不同进程，通过传递打点id可将不同应用的打点日记串连起来，根据打点时间排序。



例如，适配webmvc框架的目的是从请求头获取调用来源传递过来的打点ID(事务ID)。对DispatcherServlet#doDispatch方法插桩，从HttpServletRequest参数获取请求头“S-Tid”。“S-Tid”是自定义的请求头参数，用于传递打点ID。



笔者在实现适配webmvc和openfeign时都遇到了同样的问题，如在适配webmvc时，修改DispatcherServlet的doDispatch方法时，asm框架抛出java.lang.TypeNotPresentException。



- java.lang.TypeNotPresentException：当应用程序试图使用表示类型名称的字符串对类型进行访问，但无法找到带有指定名称的类型定义时，抛出该异常。



其原因是，使用asm框架改写DispatcherServlet类时，asm会使用Class.forName方法加载符号引用的类，如果加载不到目标类则抛出TypeNotPresentException。



默认asm会使用加载自身的类加载器去尝试加载当前改写类所依赖的一些类，而加载asm框架使用的类加载器与加载agent-core包使用的是同一个类加载器，DispatcherServlet则由SpringBoot的LaunchedClassLoader类加载器所加载。



好在ClassFileTransformer#transform方法传递了用于加载当前类的类加载器：



```
public class OnionClassFileTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className,                             Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain,                             byte[] classfileBuffer) {
             // ......
    }
}
```



- 如果当前需要改写的类是DispatcherServlet，则transform方法的第一个参数为即将用于加载DispatcherServlet类的类加载器；



我们只需要指定asm使用ClassFileTransformer#transform方法传递进来的类加载器加载DispatcherServlet依赖的类即可。



```
ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS | ClassWriter.COMPUTE_FRAMES) {
       @Override
       protected ClassLoader getClassLoader() {
            return loader;
       }
};
```



如代码所示，我们重写asm的ClassWriter类的getClassLoader方法，返回的类加载器是ClassFileTransformer#transform方法传递进来的类加载器。