---
title: dubbo-01-入门
date: 2021-03-04 09:53:30
tags:
---

# Dubbo概述

## 什么是 **PRC**?

RPC(Remote Procedure Call Protocol)——远程过程调用协议，它是一种通过网络从远 程计算机程序上请求服务，而不需要了解底层网络技术的协议。RPC 协议假定某些传输协议 的存在，如 TCP 或 UDP，为通信程序之间携带信息数据。在 OSI 网络通信模型(OSI 七层网 络模型，OSI，Open System Interconnection，开放系统互联)中，RPC 跨越了传输层和应用 层。RPC 使得开发包括网络分布式多程序在内的应用程序更加容易。

RPC 采用客户机/服务器模式(即 C/S 模式)。请求程序就是一个客户机，而服务提供程 序就是一个服务器。首先，客户机调用进程发送一个有进程参数的调用信息到服务进程，然 后等待应答信息。在服务器端，进程保持睡眠状态直到调用信息到达为止。当一个调用信息 到达，服务器获得进程参数，计算结果，发送答复信息，然后等待下一个调用信息，最后， 客户端调用进程接收答复信息，获得进程结果，然后调用执行继续进行。

## **Dubbo** 四大组件

Dubbo 中存在四大组件:

- **Provider**:服务提供者。

- **Consumer**:服务消费者。<font color="red">会从Registry下载Provider注册列表，负载均衡、限流等操作，都是Consumer自己根据这个注册列表中的Provider进行操作。</font>

- **Registry**:服务注册与发现的中心，提供目录服务，亦称为服务注册中心

- **Monitor**:统计服务的调用次数、调用时间等信息的日志服务，并可以对服务设置权限、降级处理等，称为服务管控中心



# 服务搭建

注意：业务接口已经打成jar包，消费者和生产者直接导入jar包即可。

## 直连方式

### 消费者

- pom依赖

  -  业务接口依赖

  - Dubbo 依赖(2.7.0 版本) 

  - Spring 依赖(4.3.16 版本)

  - ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
    
        <groupId>com.abc</groupId>
        <artifactId>01-consumer</artifactId>
        <version>1.0-SNAPSHOT</version>
    
        <properties>
            <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
            <maven.compiler.source>1.8</maven.compiler.source>
            <maven.compiler.target>1.8</maven.compiler.target>
            <!-- 自定义版本号 -->
            <spring-version>4.3.16.RELEASE</spring-version>
        </properties>
    
        <dependencies>
            <!--业务接口工程依赖-->
            <dependency>
                <groupId>com.abc</groupId>
                <artifactId>00-api</artifactId>
                <version>1.0-SNAPSHOT</version>
            </dependency>
    
            <!-- dubbo依赖 -->
            <dependency>
                <groupId>org.apache.dubbo</groupId>
                <artifactId>dubbo</artifactId>
                <version>2.7.0</version>
            </dependency>
            <!-- Spring依赖 -->
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-beans</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-core</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-context</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-expression</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-aop</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-aspects</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-tx</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-jdbc</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <!-- commons-logging依赖 -->
            <dependency>
                <groupId>commons-logging</groupId>
                <artifactId>commons-logging</artifactId>
                <version>1.2</version>
            </dependency>
    
        </dependencies>
    
    </project>
    ```

- 配置文件： src/main/resources 下定义 spring-consumer.xml 配置文件

  - ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    
        <!--指定当前工程在管控平台中的名称-->
        <dubbo:application name="01-consumer"/>
    
        <!--指定注册中心：不使用注册中心-->
        <dubbo:registry address="N/A"/>
    
        <!--直连式连接提供者-->
        <dubbo:reference id="someService"
                         interface="com.abc.service.SomeService"
                         url="dubbo://localhost:20880"/>
    </beans>
    ```

- main：作为测试类

  - ```java
    public class ConsumerRun {
        public static void main(String[] args) {
            ApplicationContext ac = new ClassPathXmlApplicationContext("spring-consumer.xml");
            SomeService service = (SomeService) ac.getBean("someService");
            String hello = service.hello("China");
            System.out.println(hello);
        }
    }
    ```

    

### 生产者

- pom依赖
  -  业务接口依赖
  - Dubbo 依赖(2.7.0 版本) 
  - Spring 依赖(4.3.16 版本)

- 定义接口实现类（略）

- 配置文件： src/main/resources 下定义 spring-provider.xml 配置文件

  - ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    
        <!--指定当前工程在管控平台中的名称-->
        <dubbo:application name="01-provider"/>
    
        <!--指定注册中心：不使用注册中心-->
        <dubbo:registry address="N/A"/>
    
        <!--注册业务接口实现类，它是真正的服务提供者-->
        <bean id="someService" class="com.abc.provider.SomeServiceImpl"/>
    
        <!--服务暴露-->
        <dubbo:service interface="com.abc.service.SomeService"
                       ref="someService"/>
    </beans>
    ```

- main: 启动类

  - ```java
        public static void main(String[] args) throws IOException {
            // 创建Spring容器
            ApplicationContext ac = new ClassPathXmlApplicationContext("spring-provider.xml");
            // 启动Spring容器
            ((ClassPathXmlApplicationContext) ac).start();
            // 使主线程阻塞
            System.in.read();
        }
    ```



## **Zookeeper** **注册中心**

### 消费者

- 导入依赖
  - 复制前面的提供者工程 01-provider，并更名为 02-provider-zk。修改 pom 文件，并在其中导入 Zookeeper 客户端依赖 curator。

- 配置文件

  - ```xml
    <dubbo:application name="02-consumer-zk">
            <dubbo:parameter key="qos.port" value="33333"/>
        </dubbo:application>
    
        <!--指定服务注册中心：zk单机-->
        <dubbo:registry address="zookeeper://zkOS:2181" />
        <!--<dubbo:registry protocol="zookeeper" address="zkOS:2181"/>-->
    
        <!--指定服务注册中心：zk集群-->
        <!--<dubbo:registry address="zookeeper://zkOS1:2181?backup=zkOS2:2181,zkOS3:2181,zkOS4:2181"/>-->
        <!--<dubbo:registry protocol="zookeeper" address="zkOS1:2181,zkOS2:2181,zkOS3:2181,zkOS4:2181"/>-->
    
        <dubbo:reference id="someService" check="false"
                         interface="com.abc.service.SomeService"/>
    ```

### 生产者

- 导入依赖

  - 复制前面的提供者工程 01-provider，并更名为 02-provider-zk。修改 pom 文件，并在其中导入 Zookeeper 客户端依赖 curator。

  - ```java
     <!-- zk客户端依赖：curator -->
            <dependency>
                <groupId>org.apache.curator</groupId>
                <artifactId>curator-recipes</artifactId>
                <version>2.13.0</version>
            </dependency>
            <dependency>
                <groupId>org.apache.curator</groupId>
                <artifactId>curator-framework</artifactId>
                <version>2.13.0</version>
            </dependency>
    ```

- 配置文件

  - ```xml
      <dubbo:application name="02-provider-zk"/>
    
        <!--声明注册中心：单机版zk-->
        <dubbo:registry address="zookeeper://zkOS:2181"/>
        <!--<dubbo:registry protocol="zookeeper" address="zkOS:2181"/>-->
    
        <!--声明注册中心：zk群集-->
        <!--<dubbo:registry address="zookeeper://zkOS1:2181?backup=zkOS2:2181,zkOS3:2181,zkOS4:2181"/>-->
        <!--<dubbo:registry protocol="zookeeper" address="zkOS1:2181,zkOS2:2181,zkOS3:2181,zkOS4:2181"/>-->
    
        <bean id="someService" class="com.abc.provider.SomeServiceImpl"/>
    
        <dubbo:service interface="com.abc.service.SomeService"
                ref="someService" />
    ```

  - zk集群 + 不同协议

    - ```xml
          <dubbo:application name="02-provider-zk" />
      
          <!--声明注册中心：单机版zk-->
          <dubbo:registry address="zookeeper://zkOS:2181"/>
          <!--<dubbo:registry protocol="zookeeper" address="zkOS:2181"/>-->
      
          <!--声明注册中心：zk群集-->
          <!--<dubbo:registry address="zookeeper://zkOS1:2181?backup=zkOS2:2181,zkOS3:2181,zkOS4:2181"/>-->
          <!--<dubbo:registry protocol="zookeeper" address="zkOS1:2181,zkOS2:2181,zkOS3:2181,zkOS4:2181"/>-->
      
          <bean id="someService" class="com.abc.provider.SomeServiceImpl"/>
      
          <dubbo:protocol id="dp" name="dubbo" port="20880"/>
          <dubbo:protocol id="dp2" name="dubbo" port="20881"/>
          <dubbo:protocol id="rp" name="rmi" port="9411"/>
      
          <dubbo:service interface="com.abc.service.SomeService" protocol="dp,dp2"/>
      
          <dubbo:provider id="xxx" timeout="2000" protocol="dp" default="true"/>
          <dubbo:provider id="ooo" delay="2000" protocol="dp2" default="true"/>
          <dubbo:provider id="jjj"  />
          <dubbo:provider id="kkk"  />
      ```

      

- 添加日志

  - ```properties
    log4j.appender.console=org.apache.log4j.ConsoleAppender
    log4j.appender.console.Target=System.out
    log4j.appender.console.layout=org.apache.log4j.PatternLayout
    log4j.appender.console.layout.ConversionPattern=[%-5p] %m%n
    log4j.rootLogger=info,console
    ```

    

## **将** **Dubbo** **应用到** **web** **工程**

前面所有提供者与消费者均是 Java 工程，而在生产环境中，它们都应是 web 工程，Dubbo如何应用于 Web 工程中呢？

### 生产者

- pom依赖

  -  dubbo2.7.0 版本依赖

     zk 客户端 curator 依赖

     servlet 与 jsp 依赖

     spring 相关依赖

     spring 需要的 commons-logging 依赖

     自定义 00-api 依赖

  - ```xml
    <!-- Servlet 依赖 --> 
    <dependency> 
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId> 
        <version>3.1.0</version> <scope>provided</scope>
    </dependency>
    <!-- JSP 依赖 --> 
    <dependency> 
        <groupId>javax.servlet.jsp</groupId> 
        <artifactId>javax.servlet.jsp-api</artifactId> 
        <version>2.2.1</version> <scope>provided</scope>
    </dependency>
    ```

- **定义** **web.xml**

  - ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
             version="3.1">
    
        <!--注册Spring配置文件-->
        <context-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-*.xml</param-value>
        </context-param>
    
        <!--注册ServletContext监听器-->
        <listener>
            <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
        </listener>
    
    </web-app>
    ```

- **修改** **spring-provider.xml**：略



### 消费者

- pom依赖
  -  dubbo2.7.0 版本依赖

     zk 客户端 curator 依赖

     servlet 与 jsp 依赖

     spring 相关依赖

     spring 需要的 commons-logging 依赖

     自定义 00-api 依赖

- **定义** **web.xml**

  - ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
             version="3.1">
    
        <!--对于2.6.4版本，其Spring配置文件必须指定从<context-param>中加载-->
        <!--<context-param>-->
            <!--<param-name>contextConfigLocation</param-name>-->
            <!--<param-value>classpath:spring-*.xml</param-value>-->
        <!--</context-param>-->
    
        <!--字符编码过滤器-->
        <filter>
            <filter-name>CharacterEncodingFilter</filter-name>
            <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
            <init-param>
                <param-name>encoding</param-name>
                <param-value>utf-8</param-value>
            </init-param>
            <init-param>
                <param-name>forceEncoding</param-name>
                <param-value>true</param-value>
            </init-param>
        </filter>
        <filter-mapping>
            <filter-name>CharacterEncodingFilter</filter-name>
            <url-pattern>/*</url-pattern>
        </filter-mapping>
    
        <!--注册中央调度器-->
        <servlet>
            <servlet-name>springmvc</servlet-name>
            <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
            <init-param>
                <param-name>contextConfigLocation</param-name>
                <param-value>classpath:spring-*.xml</param-value>
            </init-param>
            <load-on-startup>1</load-on-startup>
        </servlet>
        <servlet-mapping>
            <servlet-name>springmvc</servlet-name>
            <!--不能写/*，不建议写/，建议扩展名方式-->
            <url-pattern>*.do</url-pattern>
        </servlet-mapping>
    
    </web-app>
    ```

- **修改** **spring-consumer.xml**：略



# **Dubbo** **管理控制台**

2019 年初，官方发布了 Dubbo 管理控制台 0.1 版本。结构上采取了前后端分离的方式，前端使用 Vue 和 Vuetify 分别作为 Javascript 框架和 UI 框架，后端采用 Spring Boot 框架。

 **下载**

Dubbo 管理控制台的下载地址为：https://github.com/apache/incubator-dubbo-ops

**配置**

在下载的 zip 文件的解压目录的 dubbo-admin-server\src\main\resources 下，修改配置文件 application.properties。主要就是修改注册中心、配置中心，与元数据中心的 zk 地址。这是一个 springboot 工程，默认端口号为 8080，若要修改端口号，则在配置文件中增加形如 server.port=8888 的配置。

**打包**

在命令行窗口中进入到解压目录根目录，执行打包命令。mvn clean package。

打包结束后，进入到解压目录下的 dubbo-admin-distribution 目录下的 target 目录。目录下有个 dubbo-admin-0.1.jar 文件。该 Jar 包文件即为 Dubbo 管理控制台的运行文件，可以将其放到任意目录下运行。

**启动zk**

**启动管控台**

将 dubbo-admin-0.1.jar 文件存放到任意目录下，例如 D 盘根目录下，直接运行。

**访问**

在浏览器地址栏中输入 http://localhost:8080 ，即可看到 Dubbo 管理控制台界面。



# **Dubbo** **高级配置**

> 注意：consumer是从Registry获取provider注册表，所以consumer端也可以设置负载均衡等配置，从而覆盖从Registry获取的注册表
>
> | interface + version + group | provider host | 负载均衡设置 | 请求次数 |
> | --------------------------- | ------------- | ------------ | -------- |
> | someService, 1.00, beijing  | h1, h2,h3     | random       | 3        |
>
> 



## 关闭服务检查

默认情况下，若服务消费者先于服务提供者启动，则消费者端会报错。因为默认情况下消费者会在启动时查检其要消费的服务的提供者是否已经注册，若未注册则抛出异常。可以在消费者端的 spring 配置文件中添加 check=”false”属性，则可关闭服务检查功能。

## **多版本控制**

消费者和生产者都要同步改

```
<dubbo:application name="04-consumer-version"/>

<dubbo:registry address="zookeeper://zkOS:2181" />

<!--指定消费0.0.1版本，即oldService提供者-->
<!--<dubbo:reference id="someService"  version="0.0.1"-->
                 <!--interface="com.abc.service.SomeService"/>-->

<!--指定消费0.0.2版本，即newService提供者-->
<dubbo:reference id="someService"  version="0.0.2"
                 interface="com.abc.service.SomeService"/>
```

## **服务分组**

服务分组与多版本控制的使用方式几乎是相同的，只要将 version 替换为 group 即可。

但使用目的不同。使用版本控制的目的是为了升级，将原有老版本替换掉，将来不再提供老版本的服务，所以不同版本间不能出现相互调用。而分组的目的则不同，其也是针对相同接口，给出了多种实现类。但不同的是，这些不同实现并没有谁替换掉谁的意思，是针对不同需求，或针对不同功能模块所给出的不同实现。这些实现所提供的服务是并存的，所以它们间可以出现相互调用关系。例如，对于支付服务的实现，可以有微信支付实现与支付宝支付实现等。

## **同一服务支持多种协议**

这里需要理解这个服务暴露协议的意义。其是指出，消费者若要连接当前的服务，就需要通过这里指定的协议及端口号进行访问。这里的端口号可以是任意的，不一定非要使用默认的端口号（Dubbo 默认为 20880，rmi 默认为 1099）。这里指定的协议名称及端口号，在当前服务注册到注册中心时会一并写入到服务映射表中。当消费者根据服务名称查找到相应主机时，其同时会查询出消费此服务的协议、端口号等信息。其底层就是一个 Socket 编程，通过主机名与端口号进行连接。

```xml
    <dubbo:application name="05-provider-group"/>

    <dubbo:registry address="zookeeper://zkOS:2181"/>

	<dubbo:protocal name="dubbo" port="20880"/>
	<dubbo:protocal name="rmi" port="1099"/>

    <!--注册Service实现类-->
    <bean id="weixinService" class="com.abc.provider.WeixinServiceImpl"/>
    <bean id="zhifubaoService" class="com.abc.provider.ZhifubaoServiceImpl"/>

    <!--暴露服务-->
    <dubbo:service interface="com.abc.service.SomeService"
                   ref="weixinService" group="pay.weixin" protocal="dubbo,rmi"/>
    <dubbo:service interface="com.abc.service.SomeService"
                   ref="zhifubaoService" group="pay.zhifubao"/>
```

## **不同服务使用不同协议**

```xml
    <dubbo:application name="05-provider-group"/>

    <dubbo:registry address="zookeeper://zkOS:2181"/>

	<dubbo:protocal name="dubbo" port="20880"/>
	<dubbo:protocal name="rmi" port="1099"/>

    <!--注册Service实现类-->
    <bean id="weixinService" class="com.abc.provider.WeixinServiceImpl"/>
    <bean id="zhifubaoService" class="com.abc.provider.ZhifubaoServiceImpl"/>

    <!--暴露服务-->
    <dubbo:service interface="com.abc.service.SomeService"
                   ref="weixinService" group="pay.weixin" protocal="rmi"/>
    <dubbo:service interface="com.abc.service.SomeService"
                   ref="zhifubaoService" group="pay.zhifubao" protocal="dubbo"/>
```

## **负载均衡**

Dubbo 内置了四种负载均衡算法。

- random

随机算法，是 Dubbo 默认的负载均衡算法。存在服务堆积问题。

-  **roundrobin**

轮询算法。按照设定好的权重依次进行调度。

-  **leastactive** 

最少活跃度调度算法。即被调度的次数越少，其优选级就越高，被调度到的机率就越高。

- consistent hash

一致性 hash 算法。对于相同参数的请求，其会被路由到相同的提供者。

### **消费者端指定**

```
    <!--暴露服务-->
    <dubbo:ref interface="com.abc.service.SomeService"
                   ref="weixinService" group="pay.weixin" protocal="rmi" loadBalance="random"/>
```

### 服务端指定

```
    <!--暴露服务-->
    <dubbo:service interface="com.abc.service.SomeService"
                   ref="weixinService" group="pay.weixin" protocal="rmi" loadBalance="random"/>
```

