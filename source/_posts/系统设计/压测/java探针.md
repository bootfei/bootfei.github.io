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



#### 方法1：