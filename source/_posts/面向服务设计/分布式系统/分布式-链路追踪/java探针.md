---
title: java探针
date: 2021-05-13 08:21:05
tags:
---

## 原理

### 探针类SimpleAgent.java

#### 方法1：premain

```java
public class SimpleAgent {
    public static void premain(String agentArgs, Instrumentation inst) {
        ....
    }
    public static void premain(String agentArgs){
      	...
    }

}
```

premain 方法为固定写法，它有两个方法签名可以选择，JVM 首先会去寻找这个方法来启动探针，它传入了探针的参数，以及 Instrumentation 对象，该对象可以提供对字节码的修改。

```
public static void premain(String agentArgs, Instrumentation inst);
```

如果没有找到上面的方法，则会寻找

```
public static void premain(String agentArgs);
```

#### 方法2：attach

```java
public class SimpleAgent{
	  public static void agentmain(String agentArgs, Instrumentation inst) {
        System.out.println("agentmain");
    }
}
```



### 打包方式

打包为可用的 java agent 时，需要注意配置参数，上面提供了两种方式，一个是直接在`pom.xml`中指定配置

```xml
<manifestEntries>
    <Premain-Class>com.git.hui.agent.SimpleAgent</Premain-Class>
    <Agent-Class>com.git.hui.agent.SimpleAgent</Agent-Class>
    <Can-Redefine-Classes>true</Can-Redefine-Classes>
    <Can-Retransform-Classes>true</Can-Retransform-Classes>
</manifestEntries>
```

另外一个是在配置文件 `META-INF/MANIFEST.MF` 中写好(需要注意最后一个空行不可或缺）

```
Manifest-Version: 1.0
Premain-Class: com.git.hui.agent.SimpleAgent
Agent-Class: com.git.hui.agent.SimpleAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true

```



#### 方法1：使用MANIFEST.INFO

- 在资源目录(Resources)下，新建目录`META-INF`
- 在`META-INF`目录下，新建文件`MANIFEST.MF`

文件内容如下

```
Manifest-Version: 1.0
Premain-Class: com.git.hui.agent.SimpleAgent #适用于premain方式
Agent-Class: com.git.hui.agent.SimpleAgent #适用于attach方式
Can-Redefine-Classes: true
Can-Retransform-Classes: true
 #空行
```

请注意，最后的一个空行，不能少，在 idea 中，删除最后一行时，会有错误提醒

然后我们的`pom.xml`配置，需要作出对应的修改

```xml
<build>
  <plugins>
      <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-assembly-plugin</artifactId>
          <configuration>
              <descriptorRefs>
                  <descriptorRef>jar-with-dependencies</descriptorRef>
              </descriptorRefs>
              <archive>
                  <manifestFile>
                      src/main/resources/META-INF/MANIFEST.MF
                  </manifestFile>
                  <!--<manifestEntries>-->
                      <!--<Premain-Class>com.git.hui.agent.SimpleAgent</Premain-Class>-->
                      <!--<Agent-Class>com.git.hui.agent.SimpleAgent</Agent-Class>-->
                      <!--<Can-Redefine-Classes>true</Can-Redefine-Classes>-->
                      <!--<Can-Retransform-Classes>true</Can-Retransform-Classes>-->
                  <!--</manifestEntries>-->
              </archive>
          </configuration>

          <executions>
              <execution>
                  <goals>
                      <goal>attached</goal>
                  </goals>
                  <phase>package</phase>
              </execution>
          </executions>
      </plugin>
  </plugins>
</build>
```

通过`mvn assembly:assembly`命令打包



#### 方法2：pom 指定配置

在 pom.xml 文件中，添加如下配置，请注意一下`manifestEntries`标签内的参数

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
                <archive>
                    <manifestEntries>
                        <Premain-Class>com.git.hui.agent.SimpleAgent</Premain-Class>
                        <Agent-Class>com.git.hui.agent.SimpleAgent</Agent-Class>
                        <Can-Redefine-Classes>true</Can-Redefine-Classes>
                        <Can-Retransform-Classes>true</Can-Retransform-Classes>
                    </manifestEntries>
                </archive>
            </configuration>

            <executions>
                <execution>
                    <goals>
                        <goal>attached</goal>
                    </goals>
                    <phase>package</phase>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

然后通过 `mvn assembly:assembly` 命令打包，在`target`目录下，可以看到一个后缀为`jar-with-dependencies`的 jar 包，就是我们的目标



### 运行方式

创建一个web项目

```java
public class MVCWeb {
    public static void main(String[] args) throws InterruptedException {
        while(true){
          System.out.println("web running");
        	Thread.sleep(1000 * 60 * 60);
        }
    }
}
```



| 方法          | 说明                                                         | 使用姿势                                                     |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `premain()`   | agent 以 jvm 方式加载时调用，即目标应用在启动时，指定了 agent | `-javaagent:xxx.jar`                                         |
| `agentmain()` | agent 以 attach 方式运行时调用，目标应用程序正常工作后，使用attach应用 | `VirtualMachine.attach(pid)`来指定目标进程号 <br/> `vm.loadAgent("...jar")`加载 agent |



#### 方法1：使用jar-agent

web项目启动时，添加

```
-javaagent:/Users/..../target/java-agent-1.0-SNAPSHOT-jar-with-dependencies.jar
```



#### 方法2：使用attach方式

另启动一个attach项目，将agent绑定到web上

```java
public class AttachMain {
    public static void main(String[] args)
            throws IOException, AgentLoadException, AgentInitializationException, AttachNotSupportedException {
        // attach方法参数为目标应用程序的进程号
        VirtualMachine vm = VirtualMachine.attach("web的进程号");
        // 请用你自己的agent绝对地址，替换这个
        vm.loadAgent("/Users/......./target/java-agent-1.0-SNAPSHOT-jar-with-dependencies.jar");
    }
}
```





## 应用

### 使用原生JDK8开发探针统计接口耗时

#### java.lang.instrument接口定义

```java
/**
 * 注册一个Transformer，从此之后的类加载都会被Transformer拦截。
 * Transformer可以直接对类的字节码byte[]进行修改
 */
void addTransformer(ClassFileTransformer transformer);

/**
 * 对JVM已经加载的类重新触发类加载。使用的就是上面注册的Transformer。
 * retransformation可以修改方法体，但是不能变更方法签名、增加和删除方法/类的成员属性
 */
void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;

/**
 * 将一个jar加入到bootstrap classloader的 classpath里
 */
void appendToBootstrapClassLoaderSearch(JarFile jarfile);

/**
 * 获取当前被JVM加载的所有类对象
 */
Class[] getAllLoadedClasses();
```

前面两个方法比较重要，addTransformer 方法配置之后，后续的类加载都会被 Transformer 拦截。对于已经加载过的类，可以执行 retransformClasses 来重新触发这个 Transformer 的拦截。类加载的字节码被修改后，除非再次被 retransform，否则不会恢复。

通过上面的描述，可知

- 可以通过`Transformer`修改类
- 类加载时，会被触发 Transformer 拦截

#### 实现instrument.ClassFileTransformer接口

在方法的执行前，记录一个时间，执行完之后统计一下时间差，即为耗时

直接修改字节码有点麻烦，因此我们借助神器`javaassist`来修改字节码

```java
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

import java.io.ByteArrayInputStream;
import java.lang.instrument.ClassFileTransformer;
import java.security.ProtectionDomain;

public class CostTransformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
            ProtectionDomain protectionDomain, byte[] classfileBuffer) {
        // 这里我们限制下，只针对目标包下进行耗时统计
        if (!className.startsWith("com/git/hui/java/")) {
            return classfileBuffer;
        }

        CtClass cl = null;
        try {
            ClassPool classPool = ClassPool.getDefault();
            cl = classPool.makeClass(new ByteArrayInputStream(classfileBuffer));

            for (CtMethod method : cl.getDeclaredMethods()) {
                // 所有方法，统计耗时；请注意，需要通过`addLocalVariable`来声明局部变量
                method.addLocalVariable("start", CtClass.longType);
                method.insertBefore("start = System.currentTimeMillis();");
                String methodName = method.getLongName();
                method.insertAfter("System.out.println(\"" + methodName + " cost: \" + (System" +".currentTimeMillis() - start));");
            }

            byte[] transformed = cl.toBytecode();
            return transformed;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return classfileBuffer;
    }
}
```

#### SimpleAgent类注册instrument

```java
/**
 * Created by @author yihui in 16:39 20/3/15.
 */
public class SimpleAgent {

    /**
     * jvm 参数形式启动，运行此方法
     *
     * manifest需要配置属性Premain-Class
     *
     * @param agentArgs
     * @param inst
     */
    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("premain");
        customLogic(inst);
    }

    /**
     * 动态 attach 方式启动，运行此方法
     *
     * manifest需要配置属性Agent-Class
     *
     * @param agentArgs
     * @param inst
     */
    public static void agentmain(String agentArgs, Instrumentation inst) {
        System.out.println("agentmain");
        customLogic(inst);
    }

    /**
     * 统计方法耗时
     *
     * @param inst
     */
    private static void customLogic(Instrumentation inst) {
        inst.addTransformer(new CostTransformer(), true);
    }
}
```

到此 agent 完毕，打包和上面的过程一样，



#### 异常处理

```java
public class BaseMain {
    public int print(int i) {
        System.out.println("i: " + i);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return i + 2;
    }

    public void run() {
        int i = 1;
        while (true) {
            i = print(i);
        }
    }

    public static void main(String[] args) {
        BaseMain main = new BaseMain();
        main.run();
}
```

依然通过 jvm option 参数指定 agent 的方式，运行上面的代码，会发现抛异常，无法正常运行了

![img](https://user-gold-cdn.xitu.io/2020/3/17/170e67b39eaaf119?w=742&h=292&f=jpeg&s=79442)

指出了在 run 方法这里，存在字节码的错误，我们统计耗时的 Agent，主要就是在方法开始前和结束后各自新增了一行代码，我们直接补充在 run 方法中，则相当于下面的代码

![img](https://user-gold-cdn.xitu.io/2020/3/17/170e67b3a018ec29?w=672&h=159&f=jpeg&s=25739)

上面的提示很明显的告诉了，最后一行语句永远不可能达到，编译就存在异常了；

很简单，在 jvm 参数中，添加一个`-noverify` (请注意不同的 jdk 版本，参数可能不一样，我的本地是 jdk8，用这个参数；如果是 jdk7 可以试一下`-XX:-UseSplitVerifier`)

在 IDEA 开发环境下，如下配置即可

![img](https://user-gold-cdn.xitu.io/2020/3/17/170e67b39ed90844?w=716&h=226&f=jpeg&s=37619)

再次运行，正常了



#### 配合spring使用

通常来说，探针不会引入太重量级的框架，会更多地使用 `JDK` 原生的接口。然而最近发现，当探针依附在用户应用中时（ `Spring` 应用），有时难免需要使用反射调用用户接口或 `Spring` 接口，而反射调用需要类实例，使用 `Spring` 进行依赖注入的框架中，这个实例必须从 `Spring Context` 中去取。这就造成了一个问题，如何取到 `Spring Context` 呢，难道一定要在探针中引入 `Sping` 框架吗？