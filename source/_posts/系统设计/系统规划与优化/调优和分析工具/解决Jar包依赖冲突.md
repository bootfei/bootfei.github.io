---
title: 解决Jar包依赖冲突
date: 2021-05-05 19:10:53
tags:
---

https://www.sevenyuan.cn/

## 起因

应用从 jdk7 升级到 jdk8，终于可以用上新特性的语法进行代码编写，通过几轮开发、测试和验证后，在上预发环境时，应用突然无法启动，查看 tomcat 报错原因，发现是 **类转换失败 `ClassCastException`**

### 报错原因

```
Class path contains multiple SLF4J binding
23-May-2019 16:04:25.300 INFO [localhost-startStop-1] org.apache.jasper.servlet.TldScanner.scanJars At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/home/admin/xxx/WEB-INF/lib/slf4j-log4j12-1.6.1.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/admin/xxx/WEB-INF/lib/logback-classic-1.1.3.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
```

### 报错原因

```
org.slf4j.impl.Log4jLoggerFactory cannot be cast to ch.qos.logback.classic.LoggerContext
org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'cn.com.xxx.framework.log.integration.LogbackInitializer#0' defined in class path resource [spring/spring-log-init.xml]: Invocation of init method failed; nested exception is java.lang.ClassCastException: org.slf4j.impl.Log4jLoggerFactory cannot be cast to ch.qos.logback.classic.LoggerContext
    ...    
Caused by: java.lang.ClassCastException: org.slf4j.impl.Log4jLoggerFactory cannot be cast to ch.qos.logback.classic.LoggerContext
    # 出问题的加载地方
 at ch.qos.logback.ext.spring.LogbackConfigurer.initLogging(LogbackConfigurer.java:72)
 at cn.com.xxx.framework.log.integration.LogbackInitializer.init(LogbackInitializer.java:49)
 at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
 at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
 at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
 at java.lang.reflect.Method.invoke(Method.java:498)
 at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeCustomInitMethod(AbstractAutowireCapableBeanFactory.java:1706)
 at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1645)
 at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1574)
 ... 26 more
23-May-2019 15:59:12.398 SEVERE [localhost-startStop-1] org.apache.catalina.core.StandardContext.startInternal One or more listeners failed to start. Full details will be found in the appropriate container log file
```

### 查看报错代码

```
public static void initLogging(String location) throws FileNotFoundException, JoranException {
   String resolvedLocation = SystemPropertyUtils.resolvePlaceholders(location);
   URL url = ResourceUtils.getURL(resolvedLocation);
   LoggerContext loggerContext = (LoggerContext)StaticLoggerBinder.getSingleton().getLoggerFactory();
   loggerContext.reset();
   new ContextInitializer(loggerContext).configureByResource(url);
}
```

可以看到，通过 `StaticLoggerBinder.getSingleton().getLoggerFactory()` 获取 logger 上下文这段代码报错了，通过仔细定位，发现了有两个 `StaticLoggerBinder` 类

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbucAM1dSp7ycZAXYI5C2Do953p5wHIylj4Owbjn4dxhRFlKac8xbQ1JU4GYfMnqAibw68ulsq2W0ggQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**更重要的是，他们两兄弟竟然虽然不是同一个 jar 包，但是包路径和名称都一模一样！！！**

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

由于我们需要的是 `logback` 包，而不是 `slf4j-log4j12` 包，所以需要排除掉 `slf4j-log4j12` 依赖。

------

## **解决方法**

① 通过 POM 文件排查包冲突

② 安装 IDEA 的插件 `Maven Helper`

③ 定位到编译 WAR 包的 POM 文件（我们框架定义的在 Deploy 模块中）

④ 在搜索框中，输入搜索内容，点击右键可以看到选项框

- Jump To Source（跳转到源文件处）
- Exclude（排除掉）

例如我点击了 `Exclude` ，就能看到 pom 文件中，这个依赖就被排除掉了

```
<dependency>
    <groupId>cn.com.xxx</groupId>
    <artifactId>framework-conf-client</artifactId>
    <version>${xqy.framework.version}</version>
    <exclusions>
        <exclusion>
            <artifactId>slf4j-log4j12</artifactId>
            <groupId>org.slf4j</groupId>
        </exclusion>
    </exclusions>
</dependency>
```

排除依赖后，提交代码，重新打包，部署一条龙，顺利启动~

------

## **思考**

包冲突解决是简单的，通过 maven 插件可以精确找到依赖，然后进行 Exclude，可是在本地开发、测试环境都没有出现的问题，却在预发环境出现了，所以排除了业务逻辑代码的原因，简单考虑了几个因素和原因：

- jdk 版本
- tomcat 版本
- 类加载机制
- 第三方 jar 互相依赖

由于 jdk 和 tomcat 这两者没有明显的报错原因，所以先去排查类的加载机制

------

### 类加载机制

我们写的 Java 应用代码，一般是通过 `App ClassLoader` 应用加载器进行加载，它不会自己先去加载它，而是通过 `Extension ClassLoader` 扩展类加载器进行加载（其中扩展类加载器又会去找 `Bootstrap ClassLoader` 启动类加载器进行加载），只有父加载器无法加载情况下，才会让下级加载器进行加载。

------

### ClassLoader

Java 使用的是双亲委派加载机制，通过查看 `ClassLoader` 类，可以对此有所了解。

类被成功加载后，将被放入到内存中，内存中存放 Class 实例对象。

```
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        // 首先，检查 class 是否已经被加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            // 如果没有被加载
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    // 寻找 parent 加载器
                    c = parent.loadClass(name, false);
                } else {
                    // 如果父加载器不存在，则委托给启动类加载器加载
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                // 如果仍然无法加载，才会尝试自身加载
                long t1 = System.nanoTime();
                c = findClass(name);
                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

------

### 类加载顺序

从代码中了解到，如果某个名字的类 <!--名字 = 类路径+类名称--> 被加载后，类加载器是不会再重新加载，所以我们的问题根本原因可以是出现在：

**先加载了 `org.slf4j` 包的 `org.slf4j.impl.StaticLoggerBinder`，同名的 `ch.qos.logback` 包下的 `StaticLoggerBinder` 类没有被加载**

> 跟JAR文件的文件名有关。按照字母的顺序加载JAR文件。有了这个类以后，后面的类则不会加载了。
>
> jvm 加载包名和类名相同的类时，先加载classpath中jar路径放在前面的，包名类名都相同，那jvm没法区分了，如果使用ide一般情况下是会提示发生冲突而报错，若不报错，只有第一个包被引入（在classpath路径下排在前面的包），第二个包会在classloader加载类时判断重复而忽略。

------

### 查看加载顺序

在 jvm 启动脚本中，添加 `-verbose` 参数或者 `-XX:+TraceClassLoading`

```
[Loaded java.lang.CloneNotSupportedException from /Users/jingqi/.jrebel/bootcache/jrebel-bootstrap-89f567048120ef32b965233b5ba2f7ca.jar]
[Loaded java.lang.Thread$State from /Users/jingqi/.jrebel/bootcache/jrebel-bootstrap-89f567048120ef32b965233b5ba2f7ca.jar]
[Loaded java.util.TreeMap$NavigableSubMap from /Users/jingqi/.jrebel/bootcache/jrebel-bootstrap-89f567048120ef32b965233b5ba2f7ca.jar]
[Loaded java.util.TreeMap$AscendingSubMap from /Users/jingqi/.jrebel/bootcache/jrebel-bootstrap-89f567048120ef32b965233b5ba2f7ca.jar]
[Loaded java.util.TreeMap$NavigableSubMap$EntrySetView from /Users/jingqi/.jrebel/bootcache/jrebel-bootstrap-89f567048120ef32b965233b5ba2f7ca.jar]
[Loaded java.util.TreeMap$AscendingSubMap$AscendingEntrySetView from /Users/jingqi/.jrebel/bootcache/jrebel-bootstrap-89f567048120ef32b965233b5ba2f7ca.jar]
[Loaded java.util.TreeMap$NavigableSubMap$SubMapIterator from /Users/jingqi/.jrebel/bootcache/jrebel-bootstrap-89f567048120ef32b965233b5ba2f7ca.jar]
[Loaded java.util.TreeMap$NavigableSubMap$SubMapEntryIterator from /Users/jingqi/.jrebel/bootcache/jrebel-bootstrap-89f567048120ef32b965233b5ba2f7ca.jar]
```

之前在本地开发中，IDEA 优化先加载了 `ch.qos.logback` 的 `StaticLoggerBinder` 类，然后后面的 `org.slf4j` 包下的同名类就没有被加载。

但这样也有个不明白，按理说加载顺序按照**字母顺序**加载，预发环境还是能够跟本地开发一样，加载到我们需要的类。实际上，加载器加载到的是另一个类，导致应用无法启动。

> 问题就是jar的加载顺序问题，而这个顺序实际上是由文件系统决定的，linux内部是用inode来指示文件的。
>
> 这种储存文件元信息的区域就叫做inode，中文译名为”索引节点”。每一个文件都有对应的inode，里面包含了与该文件有关的一些信息。
>
> Unix/linux系统内部不使用文件名，而使用inode号码来识别文件。对于系统来说，文件名只是inode号码便于识别的别称或者绰号。

为了验证 `inode` 是否是问题的原因，我做了以下测试：

------

### inode 测试加载顺序

#### 本地 Tomcat8 测试（正常启动）

将之前在 uat 环境有问题的代码版本重新打包，不使用 idea 工具，直接用 tomcat8 启动，并且在 `catalina.sh` 脚本中加入类加载打印参数 `-XX:+TraceClassLoading`

catalina.sh

```
# Register custom URL handlers
# Do this here so custom URL handles (specifically 'war:...') can be used in the security policy
JAVA_OPTS="$JAVA_OPTS -XX:+TraceClassLoading"
```

查看 `catalina.out` 输入日志，发现先加载的是 logback 包中 `StaticLoggerBinder`

在 `WEB-INF/lib` 下比较 inode 大小（正常解压和启动 logback < slf4j)

```
ll -i logback-classic-1.1.3.jar slf4j-log4j12-1.6.1.jar
34153162 -rw-r-----  1 jingqi  staff   274K  8  1  2018 logback-classic-1.1.3.jar
34153180 -rw-r-----  1 jingqi  staff   9.5K 10 17  2018 slf4j-log4j12-1.6.1.jar
```

------

#### 本地 Tomcat8 测试（删包，先添加 slf4j，后添加 logback）

- 清理掉 catalina.out
- 重新上传包
- 比较 inode 大小
- 重新启动，查看类加载日志

**比较 inode 大小（发现 slf4j < logback)**

```
# ll -i logback-classic-1.1.3.jar slf4j-log4j12-1.6.1.jar
34162396 -rw-r--r--  1 jingqi  staff   274K  8  1  2018 logback-classic-1.1.3.jar
34162361 -rw-r--r--  1 jingqi  staff   9.5K 10 17  2018 slf4j-log4j12-1.6.1.jar
```

重新启动后，查看 `catalina.out` 日志，发现类加载顺序与之前的一致，应用也能正常启动，所以本地开发无法复现 =-=

------

#### 在 uat 环境服务器测试

在 `WEB-INF/lib` 路径下，先将这两个包删掉，尝试有不同的上传顺序，模拟 tomcat 解压 war 包

```
[admin@uat-96-0-248 lib]$ rm logback-classic-1.1.3.jar  slf4j-log4j12-1.6.1.jar
[admin@uat-96-0-248 lib]$ rz
[admin@uat-96-0-248 lib]$ # Received /Users/jingqi/Downloads/slf4j-log4j12-1.6.1.jar
[admin@uat-96-0-248 lib]$ rz
[admin@uat-96-0-248 lib]$ # Received /Users/jingqi/Downloads/logback-classic-1.1.3.jar
# 第一次上传顺序 1、slf4j-log4j12-1.6.1.jar 2、logback-classic-1.1.3.jar
# inode 比较：slf4j < logback
[admin@uat-96-0-248 lib]$ ll -i logback-classic-1.1.3.jar slf4j-log4j12-1.6.1.jar
396731 -rw-r--r-- 1 admin admin 280928 8月   1 2018 logback-classic-1.1.3.jar
394075 -rw-r--r-- 1 admin admin   9753 10月 17 2018 slf4j-log4j12-1.6.1.jar
[admin@uat-96-0-248 lib]$ rm logback-classic-1.1.3.jar  slf4j-log4j12-1.6.1.jar
[admin@uat-96-0-248 lib]$ rz
[admin@uat-96-0-248 lib]$ # Received /Users/jingqi/Downloads/logback-classic-1.1.3.jar
[admin@uat-96-0-248 lib]$ rz
[admin@uat-96-0-248 lib]$ # Received /Users/jingqi/Downloads/slf4j-log4j12-1.6.1.jar
# 第二次上传顺序 1、logback-classic-1.1.3.jar 2、slf4j-log4j12-1.6.1.jar
# inode 比较：logback < slf4j
[admin@uat-96-0-248 lib]$ ll -i logback-classic-1.1.3.jar slf4j-log4j12-1.6.1.jar
394075 -rw-r--r-- 1 admin admin 280928 8月   1 2018 logback-classic-1.1.3.jar
396731 -rw-r--r-- 1 admin admin   9753 10月 17 2018 slf4j-log4j12-1.6.1.jar
```

分别测试了两种场景，发现只要这两个包都存在的情况下，无论 `inode` 两者的大小，都是先加载了 `slf4j` 包的类，导致启动报错

![Image](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

------

### 测试结束

通过多种测试场景，发现本地开发、测试环境都无法复现的问题，在 uat 环境下，只要这两个包同时存在，都会启动报错

最后在官方文档发现这个：

> The order in which the JAR files in a directory are enumerated in the expanded class path is not specified and may vary from platform to platform and even from moment to moment on the same machine. A well-constructed application should not depend upon any particular order. If a specific order is required, then the JAR files can be enumerated explicitly in the class path.

大意为：同一个目录下，jvm加载jar包顺序是无法保证的，每个系统的都不一样，甚至同一个系统不同的时刻加载都不一样。

于是乎，我也不纠结某台服务器上的类加载顺序，在开发阶段就先将这个包冲突的情况，给提前解决掉~

------

## **总结**

### 冲突提示信息

- **java.lang.ClassNotFoundException**：类型转换错误，这个报错跟我这次遇到的一样，本应该引入的是 `logback` 包的类，但是实际引入的是 `slf4j` 下的同名类，导致类型转换错误
- **java.lang.NoSuchMethodError**：找不到特定方法，如果有两个同名的包但是不同版本，例如 xxx-1.1和 xxx-1.2包同时存在，先加载了 1.1 版本的类，但是 1.2 版本中才提供了新方法，导致提示找不到特定方法
- **java.lang.NoClassDefFoundError，java.lang.LinkageError**

### 排查思路

1、查看 `catalina.sh` 堆栈信息，找到有问题的类

2、通过 IDEA ，在打包的 POM 文件中，使用 `Maven Helper` 插件找出冲突的依赖，确定项目需要的 jar 包，`Exclude` 掉不需要的依赖。

### 提前预防

**1、使用工具检查依赖冲突**

冲突检测插件 ：`maven-enforcer-plugin`

引用新的第三方依赖（工具包或者框架包），通过 Maven 插件检查一下 conflict 依赖，提前进行 Exclude

**2、统一服务器版本**

在测试阶段，准备好和生产环境一样的服务器，提前进行测试，避免依赖冲突的 `WAR` 包上传到生产环境，例如我们有一台 UAT 服务器，与生产环境一样配置，提前测试

------

