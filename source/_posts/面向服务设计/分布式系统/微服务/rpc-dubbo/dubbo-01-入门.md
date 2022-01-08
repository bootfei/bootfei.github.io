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

## **集群容错策略与重试次数**

集群容错指的是，当消费者调用提供者集群时发生异常的处理方案。

Dubbo 内置了 6 种集群容错策略。

 **Failover** 

故障转移策略。当消费者调用提供者集群中的某个服务器失败时，其会自动尝试着调用

其它服务器。该策略通常用于读操作，例如，消费者要通过提供者从 DB 中读取某数据。但

重试会带来服务延迟。

 **Failfast** 

快速失败策略。消费者端只发起一次调用，若失败则立即报错。通常用于非幂等性的写

操作，比如新增记录。

幂等：在请求参数相同的前提下，请求一次与请求 n 次，对系统产生的影响是相同的。

 GET：幂等

 POST：非幂等

 PUT：幂等

 DELETE：幂等

**Failsafe** 

失败安全策略。当消费者调用提供者出现异常时，直接忽略本次消费操作。该策略通常用于执行相对不太重要的服务，例如，写入审计日志等操作。

 **Failback** 

失败自动恢复策略。消费者调用提供者失败后，Dubbo 会记录下该失败请求，然后定时

自动重新发送该请求。该策略通常用于实时性要求不太高的服务，例如消息通知操作。

**Forking** 

并行策略。消费者对于同一服务并行调用多个提供者服务器，只要一个成功即调用结束

并返回结果。通常用于实时性要求较高的读操作，但其会浪费较多服务器资源。

**Broadcast** 

广播策略。广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有

提供者更新缓存或日志等本地资源信息。



### **消费者端指定**

```xml
    <!--暴露服务-->
    <dubbo:ref interface="com.abc.service.SomeService"
                   ref="weixinService" group="pay.weixin" cluster="failfast" reties="2"/>
```

### 服务端指定

```xml
    <!--暴露服务-->
    <dubbo:service interface="com.abc.service.SomeService"
                   ref="weixinService" group="pay.weixin" cluster="failfast" reties="2"/>
```

## **服务降级基础（面试题）**

 **什么是服务降级**

服务降级，当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务有策略的降低服务级别，以释放服务器资源，保证核心任务的正常运行。

**服务降级方式**

能够实现服务降级方式很多，常见的有如下几种情况：

 部分服务暂停	

 全部服务暂停

 随机拒绝服务

 部分服务延迟

**服务降级与** **Mock** **机制**

Dubbo的服务降级采用的是mock机制。其具有两种降级处理方式：Mock Null降级处理，与 Mock Class 降级处理。

### Mock Null降级处理

只需要修改服务端

- **修改** **pom** **文件**：由于这里不再需要 00-api 工程了，所以在 pom 文件中将对 00-api 工程的依赖删除即可。改为消费者和提供者在自己的项目中保留一份相同的接口

  - ```java
    //consumer自己保留的接口，provider也是如此
    package com.abc.service;
    
    public interface UserService {
        String getUsernameById(int id);
        void addUser(String username);
    }
    ```

    

- **修改** **spring-consumer.xml**: mock="return null"

  - ```xml
        <dubbo:reference id="userService" mock="return null" check="false"
                         interface="com.abc.service.UserService"/>
    ```

- **修改消费者启动类**

  - ```java
    public static void main(String[] args) {
        ApplicationContext ac = new ClassPathXmlApplicationContext("spring-consumer.xml");
        UserService service = (UserService) ac.getBean("userService");
    
        // 对于有返回值的方法，其返回结果为null
        String username = service.getUsernameById(3);
        System.out.println("username = " + username);
        // 对于没有返回值的方法，其没有任何结果
        service.addUser("China");
    }
    ```

    

### Mock Class 降级处理

降级都是指的调用端(consumer)，所以是consumer需要修改

- **定义** **Mock Class**

  - 在业务接口所在的包中，本例为 com.abc.service 包，定义一个类，该类的命名需要满足以下规则：业务接口简单类名 + Mock

  - ```java
    package com.abc.service;
    
    public class UserServiceMock implements UserService {
    
        @Override
        public String getUsernameById(int id) {
            return "没有该用户：" + id;
        }
    
        @Override
        public void addUser(String username) {
            System.out.println("添加该用户失败：" + username);
        }
    }
    ```

- **修改** **spring-consumer.xml**: mock="true"

  - ```xml
        <dubbo:reference id="userService" mock="true" check="false"
                         interface="com.abc.service.UserService"/>
    ```

    



## **服务调用超时**

前面的服务降级的发生，其实是由于消费者调用服务超时引起的，即从发出调用请求到获取到提供者的响应结果这个时间超出了设定的时限。默认服务调用超时时限为 1 秒。可以在消费者端与提供者端设置超时时限。

### 生产者

- **修改依赖**：由于这里不再需要 00-api 工程了，所以在 pom 文件中将对 00-api 工程的依赖删除即可。因为provider and consumer share the same interface in their responding projecct.
- **定义接口实现类**
  - 在 com.abc.provider 包中定义接口的实现类。该实现类中的业务方法添加一个 2 秒的Sleep，以延长向消费者返回结果的时间。

### 消费者

- 配置类：timeout="2000"

  - ```xml
        <dubbo:reference id="userService" mock="true" timeout="2000"
                         interface="com.abc.service.UserService"/>
    ```



## 服务限流

> - **直接限流**
>
>   - *executes** **限流** **–** **仅提供者端**
>
>     - 该属性仅能设置在提供者端。可以设置为接口级别，也可以设置为方法级别。限制的是服务（方法）并发执行数量。execute="10"
>
>     - ```
>           <dubbo:reference ref="userService"  execute="10"
>                            interface="com.abc.service.UserService"/>
>       ```
>
>   - **accepts** **限流** **–** **仅提供者端**
>
>     - 该属性仅可设置在提供者端的<dubbo:provider/>与<dubbo:protocol/>。用于对指定协议的连接数量进行限制。
>
>     - ```xml
>       限制当前提供者在使用dubbo协议时最多接受10个消费者链接
>       <dubbo:provider protocal="dubbo" accepts=10></dubbo:provider>
>       
>       限制当前提供者在使用dubbo协议时最多接受10个消费者链接
>       <dubbo:protocal name="dubbo" port="20880" accepts=10></dubbo:protocal>
>       ```
>
>   - **actives** **限流** **–** **两端**
>
>     - 该限流方式与前两种不同的是，其可以设置在提供者端，也可以设置在消费者端。可以设置为接口级别，也可以设置为方法级别。
>
>     - **提供者端限流**
>
>       根据消费者与提供者间建立的连接类型的不同，其意义也不同：
>
>        长连接：表示当前长连接最多可以处理的请求个数。与长连接的数量没有关系。
>
>        短连接：表示当前服务可以同时处理的短连接数量。
>
>       - ```xml
>         <dubbo:service interface="com.abc.service.SomeService" ref="someService" actives="10"/>
>         
>         <dubbo:service interface="com.abc.service.SomeService" ref="someService">
>         	<dubbo:method name="hello" actives="10"></dubbo:method>
>         </dubbo:service>
>                            
>         ```
>
>     - **消费者端限流**
>
>       根据消费者与提供者间建立的连接类型的不同，其意义也不同：
>
>        长连接：表示当前消费者所发出的长连接中最多可以提交的请求个数。与长连接的数量没有关系。
>
>        短连接：表示当前消费者可以提交的短连接数量。
>
>       - ```xml
>         <dubbo:ref interface="com.abc.service.SomeService" id="someService" actives="10"/>
>         
>         <dubbo:ref interface="com.abc.service.SomeService" id="someService">
>         	<dubbo:method name="hello" actives="10"></dubbo:method>
>         </dubbo:service>
>         ```
>
>   - **connections** **限流 - 两端** 
>
>     - 可以设置在提供者端，也可以设置在消费者端。限定连接的个数。对于短连接，该属性效果与 actives 相同。但对于长连接，其限制的是长连接的个数。一般情况下，我们会使 connectons 与 actives 联用，让 connections 限制长连接个数，让actives 限制一个长连接中可以处理的请求个数。联用前提：使用默认的 Dubbo 服务暴露协议。
>
>     - **提供者端限流**
>
>       - ```xml
>         <dubbo:service interface="com.abc.service.SomeService" ref="someService" connections="10"/>
>         
>         <dubbo:service interface="com.abc.service.SomeService" ref="someService">
>         	<dubbo:method name="hello" connections="10"></dubbo:method>
>         </dubbo:service>
>         ```
>
>     - **消费者端限流**
>
>       - ```xml
>         <dubbo:ref interface="com.abc.service.SomeService" id="someService" connections="10"/>
>         
>         <dubbo:ref interface="com.abc.service.SomeService" id="someService">
>         	<dubbo:method name="hello" connections="10"></dubbo:method>
>         </dubbo:service>
>         ```
>
> - **间接限流**
>
>   - **延迟连接 **– **仅消费者端**
>
>     - 仅可设置在消费者端，且不能设置为方法级别。仅作用于 Dubbo 服务暴露协议。将长连接的建立推迟到消费者真正调用提供者时。可以减少长连接的数量。
>
>     - ```xml
>       //消费者端该接口的所有方法都是延迟建立连接
>       <dubbo:ref interface="com.abc.service.SomeService" id="someService" lazy="true"/>
>       
>       //消费者端所有接口的所有方法都是延迟建立连接
>       <dubbo:consumer lazy="true"></dubbo:consumer>
>       ```
>
>   - **粘连连接** **–** **仅消费者**
>
>     - 仅能设置在消费者端，其可以设置为接口级别，也可以设置为方法级别。仅作用于Dubbo 服务暴露协议。其会使客户端尽量向同一个提供者发起调用，除非该提供者挂了，其会连接另一台。只要启用了粘连连接，其就会自动启用延迟连接。其限制的是流向，而非流量。
>
>     - ```xml
>       <dubbo:ref interface="com.abc.service.SomeService" id="someService" sticky="true"/>
>       
>       <dubbo:ref interface="com.abc.service.SomeService" id="someService">
>       	<dubbo:method name="hello" connections="10" sticky="true"></dubbo:method>
>       </dubbo:service>
>       ```
>
>   - **负载均衡 - 双端**
>
>     - 可以设置在消费者端，亦可设置在提供者端；可以设置在接口级别，亦可设置在方法级别。其限制的是流向，而非流量。
>
>     - ```
>       loadBalance="leastactive"
>       ```
>
>       

## **声明式缓存** - 仅消费者

为了进一步提高消费者对用户的响应速度，减轻提供者的压力，Dubbo 提供了基于结果的声明式缓存。该缓存是基于消费者端的，所以使用很简单，只需修改消费者配置文件，与提供者无关。

- **修改消费者配置文件**：仅需在<dubbo:reference/>中添加 cache=”true”属性即可。
- **默认缓存** **1000** **个结果**: 声明式缓存中可以缓存多少个结果呢？默认可以缓存 1000 个结果。若超出 1000，将采用 LRU 策略来删除缓存，以保证最热的数据被缓存。注意，该删除缓存的策略不能修改。
- **应用场景**: 应用于查询结果不会发生改变的情况，例如，查询某产品的序列号、订单、身份证号等。



## 多注册中心

- 消费者

  - 配置文件

    - ```xml
          <!--声明注册中心-->
          <dubbo:registry id="bjCenter" address="zookeeper://bjZK:2181"/>
          <dubbo:registry id="gzCenter" address="zookeeper://gzZK:2181"/>
          <dubbo:registry id="cqCenter" address="zookeeper://cqZK:2181"/>
      
          <!--指定调用bjCenter注册中心微信服务-->
          <dubbo:reference id="weixin"  group="pay.weixin" registry="bjCenter"
                           interface="com.abc.service.SomeService"/>
      
          <!--指定调用gzCenter与cqCenter注册中心支付宝服务-->
          <dubbo:reference id="gzZhifubao"  group="pay.zhifubao" registry="gzCenter"
                           interface="com.abc.service.SomeService"/>
          <dubbo:reference id="cqZhifubao"  group="pay.zhifubao" registry="cqCenter"
                           interface="com.abc.service.SomeService"/>
      
      ```

- 生产者

  - 配置文件

    - ```xml
          <!--声明注册中心-->
          <dubbo:registry id="bjCenter" address="zookeeper://bjZK:2181"/>  <!--北京中心-->
          <dubbo:registry id="shCenter" address="zookeeper://shZK:2181"/>  <!--上海中心-->
          <dubbo:registry id="gzCenter" address="zookeeper://gzZK:2181"/>  <!--广州中心-->
          <dubbo:registry id="cqCenter" address="zookeeper://cqZK:2181"/>  <!--重庆中心-->
      
          <!--注册Service实现类-->
          <bean id="weixinService" class="com.abc.provider.WeixinServiceImpl"/>
          <bean id="zhifubaoService" class="com.abc.provider.ZhifubaoServiceImpl"/>
      
          <!--暴露服务：同一个服务注册到不同的中心；不同的服务注册到不同的中心-->
          <dubbo:service interface="com.abc.service.SomeService"
                         ref="weixinService" group="pay.weixin" register="bjCenter, shCenter"/>
          <dubbo:service interface="com.abc.service.SomeService"
                         ref="zhifubaoService" group="pay.zhifubao" register="gzCenter, cqCenter"/>
      ```

      

## **单功能注册中心** -- 仅提供者，但是提供者和消费者是相对概念

注册中心提供服务发现、服务注册两种功能，但是某些场景下，我们只想用其中一个功能。

这些仅订阅或仅注册，只对当前配置文件中的服务起作用，不会影响注册中心本身的功能。

- **仅订阅**

  - 概念: 对于某服务来说，其可以发现和调用注册中心中的其它服务，但不能被其它服务发现和调用，这种情形称为仅订阅。简单说就是，仅可去发现，但不能被发现。其底层的实现是，当前服务可以从注册中心下载注册列表，但其不会将自己的信息写入到注册列表。

  - **设置方式**：对于“仅订阅”注册中心的实现，只需修改提供者配置文件，在<dubbo:registry/>标签中添加 register=”false”属性。即对于当前服务来说，注册中心不再接受其注册，但该服务可以通过注册中心去发现和调用其它服务。

- **仅注册**

  - 概念：对于某服务来说，其可以被注册中心的其它服务发现和调用，但不能发现和调用注册中心中的其它服务，这种情形称为仅注册。简单来说就是，仅可被发现，但不能去发现。[从底层实现来说就是，当前服务可以写入到注册列表，但其不能下载注册列表。]()
  - 设置方式：对于“仅注册”注册中心的实现，[只需修改提供者配置文件]()，在<dubbo:registry/>标签中添加 subscribe=”false”的属性。即对于当前服务来说，注册中心中的其它服务可以发现和调用当前服务，但其不能发现和调用其它服务。




## **服务暴露延迟 -- 仅提供者**

如果我们的服务启动过程需要 warmup 事件，就可以使用 delay 进行服务延迟暴露。只需在服务提供者的<dubbo:service/>标签中添加 delay 属性。其值可以有三类：

 正数：单位为毫秒，表示在提供者对象创建完毕后的指定时间后再发布服务。

 0：默认值，表示当前提供者创建完毕后马上向注册中心暴露服务。

 -1：表示在 Spring 容器初始化完毕后再向注册中心暴露服务。

> [先提供者创建完成，然后Spring容器初始化完成]()



## 消费者的异步调用

在 Dubbo 简介时，我们分析了 Dubbo 的四大组件工作原理图，其中消费者调用提供者采用的是同步调用方式。其实，消费者对于提供者的调用，也可以采用异步方式进行调用。异步调用一般应用于提供者提供的是耗时性 IO 服务。

比如consumer 需要同时调用 provider 的a服务消耗3ms，b服务5ms

- 同步的话：消耗=3+5
- 异步的话：消耗=min（3，5）

### Future异步执行原理  -- 仅消费者

异步方法调用执行原理如下图所示，其中实线为同步调用，而虚线为异步调用。

![img](https://img-blog.csdnimg.cn/20190803211841665.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI5NjUyMDM=,size_16,color_FFFFFF,t_70)

 UserThread：消费者线程

 IOThrea：提供者线程

 Server：对 IO 型操作的真正执行者

> get/wait方法时阻塞的

#### 提供者 -- 不需要做任何修改

#### 消费者

- 配置文件

  - Third, Fourth异步方式

  - ```
        <dubbo:application name="10-consumer-async"/>
    
        <dubbo:registry address="zookeeper://zkOS:2181" />
    
        <dubbo:reference id="otherService"  timeout="20000"
                         interface="com.abc.service.OtherService" >
            <dubbo:method name="doThird" async="true"/>
            <dubbo:method name="doFourth" async="true"/>
        </dubbo:reference>
    ```

- 测试类

  - 测试异步调用的请求时间

    - 请求非常快，但是没数据

    - ```
        public static void main(String[] args)
                  throws ExecutionException, InterruptedException {
              ApplicationContext ac =
                      new ClassPathXmlApplicationContext("spring-consumer.xml");
              OtherService service = (OtherService) ac.getBean("otherService");
      
              // 记录异步调用开始时间
              long asyncStart = System.currentTimeMillis();
      
              // 异步调用
              service.doThird();
              service.doFourth();
      
              long syncInvokeTime = System.currentTimeMillis() - asyncStart;
              System.out.println("两个异步调用共计用时（毫秒）：" + syncInvokeTime);
          }
      ```

  - 测试异步调用，获取结果的时间

    - 就比较慢，等待结果返回

    - 注意：RpcContext.getContext().getFuture()是和异步调用成对出现的

      - ```java
        String result1 = service.doThird();
        System.out.println("调用结果1 = " + result1);
        Future<String> thirdFuture = RpcContext.getContext().getFuture();
        
        String result3 = service.doFourth();
        System.out.println("调用结果3 = " + result3);
        Future<String> fourFuture = RpcContext.getContext().getFuture();
        ```

      - 错误写法是: 导致thirdFuture和fourFuture都是最近的调用service.doFourth()的结果

        ```java
        String result1 = service.doThird();
        String result3 = service.doFourth();
        
        System.out.println("调用结果1 = " + result1);
        Future<String> thirdFuture = RpcContext.getContext().getFuture();
        
        System.out.println("调用结果3 = " + result3);
        Future<String> fourFuture = RpcContext.getContext().getFuture();
        ```

        

    - ```java
      public static void main(String[] args)
                  throws ExecutionException, InterruptedException {
              ApplicationContext ac =
                      new ClassPathXmlApplicationContext("spring-consumer.xml");
              OtherService service = (OtherService) ac.getBean("otherService");
      
              // 记录异步调用开始时间
              long asyncStart = System.currentTimeMillis();
      
              // 异步调用
              String result1 = service.doThird();
              System.out.println("调用结果1 = " + result1);
              Future<String> thirdFuture = RpcContext.getContext().getFuture();
      
              String result3 = service.doFourth();
              System.out.println("调用结果3 = " + result3);
              Future<String> fourFuture = RpcContext.getContext().getFuture();
      
              // 阻塞
              String result2 = thirdFuture.get();
              System.out.println("调用结果2 = " + result2);
              String result4 = fourFuture.get();
              System.out.println("调用结果4 = " + result4);
      
              long useTime = System.currentTimeMillis() - asyncStart;
              System.out.println("获取到异步调用结果共计用时：" + useTime);
          }
      ```

      

### **CompletableFuture** **异步调用**  -- 消费者和生产者

使用 Future 实现异步调用，对于无需获取返回值的操作来说不存在问题，但消费者若需要获取到最终的异步执行结果，则会出现问题：消费者在使用 Future 的 get()方法获取返回值时被阻塞, CPU被无意义的轮休消耗。

为了解决这个问题，Dubbo 又引入了 CompletableFuture 来实现对提供者的异步调用。

#### 消费者

- 配置文件去除asyn属性

- 公共接口类

  - 以前是

    ```
    public interface OtherService {
        String doFirst();
        String doSecond();
        String doThird();
        String doFourth();
    }
    ```

    

  - 现在是

    ```
    public interface OtherService {
        String doFirst();
        String doSecond();
    
        CompletableFuture<String> doThird();
        CompletableFuture<String> doFourth();
    }
    ```

    

- 主函数使用CompletableFuture

  - ```java
     public static void main(String[] args)
                throws ExecutionException, InterruptedException {
            ApplicationContext ac =
                    new ClassPathXmlApplicationContext("spring-consumer.xml");
            OtherService service = (OtherService) ac.getBean("otherService");
    
            // 记录异步调用开始时间
            long asyncStart = System.currentTimeMillis();
    
            // 异步调用
            CompletableFuture<String> doThirdFuture = service.doThird();
            CompletableFuture<String> doFourthFuture = service.doFourth();
    
            long syncInvokeTime = System.currentTimeMillis() - asyncStart;
            System.out.println("两个异步调用共计用时（毫秒）：" + syncInvokeTime);
    
            // 回调方法
            doThirdFuture.whenComplete((result, throwable) -> {
                if(throwable != null) {
                    throwable.printStackTrace();
                } else {
                    System.out.println("异步调用提供者的doThird()返回值：" + result);
                }
            });
    
            doFourthFuture.whenComplete((result, throwable) -> {
                if(throwable != null) {
                    throwable.printStackTrace();
                } else {
                    System.out.println("异步调用提供者的doFourth()返回值：" + result);
                }
            });
    
            long getResultTime = System.currentTimeMillis() - asyncStart;
            System.out.println("=============（毫秒）：" + getResultTime);
    
        }
    ```

    

#### 提供者

- 和consumer一样，把公共接口类改了

  - 以前具体的接口实现

    ```
    @Override
    public String doThird() {
    	sleep();
    	return "doThird()";
    }	
    ```

    

  - 现在具体的接口实现

    ```java
     @Override
        public CompletableFuture<String> doThird() {
            long startTime = System.currentTimeMillis();
            // 耗时操作仍由业务线程调用
            sleep();
            CompletableFuture<String> future =
                    CompletableFuture.completedFuture("doThird()-----");
            long endTime = System.currentTimeMillis();
            long useTime = endTime - startTime;
            System.out.println("doThird()方法执行用时：" + useTime);
            return future;
        }
    ```



#### **总结**

Future 与 CompletableFuture 的对比：

-  Future：Dubbo2.7.0 版本之前消费者异步调用提供者的实现方式。源自于 JDK5，对异步结果的获取采用了阻塞与轮询方式。

-  CompletableFuture：Dubbo2.7.0 版本之后消费者异步调用提供者的实现方式。源自于JDK8，对异步结果的获取采用了回调的方式。

[介绍Future与CompletableFuture的文章](https://deepakvadgama.com/blog/completable-future-internals/)



## **提供者的异步执行**

从前面“对提供者的异步调用”例子可以看出，消费者对提供者实现了异步调用，消费者线程的执行过程不再发生阻塞，但提供者对 IO 耗时操作仍采用的是同步调用，即 IO 操作仍会阻塞 Dubbo 的提供者线程。

> 但需要注意，提供者对 IO 操作的异步调用，并不会提升 RPC 响应速度，因为耗时操作终归是需要消耗那么多时间后才能给出结果的。
>
> 对用户体验没什么提升，就是接口延迟；但是极大的提升了吞吐量，不再阻塞业务线程。

- 以前接口

  ```java
  @Override
  public CompletableFuture<String> doThird() {
      long startTime = System.currentTimeMillis();
      // 耗时操作仍由业务线程调用, 所以阻塞了业务线程
      sleep();
      CompletableFuture<String> future =
          CompletableFuture.completedFuture("doThird()-----");
      long endTime = System.currentTimeMillis();
      long useTime = endTime - startTime;
      System.out.println("doThird()方法执行用时：" + useTime);
      return future;
  }
  ```

- 现在具体的接口实现:<font color="red"> 耗时操作不再由业务线程直接调用</font>

  ```java
   @Override
      public CompletableFuture<String> doThird() {
          long startTime = System.currentTimeMillis();
          // 异步调用耗时操作
          CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
              // 耗时操作是由CompletableFuture调用的，而不是由业务线程直接调用，所以不再阻塞业务线程
              sleep();
              return "doThird()";
          });
          long endTime = System.currentTimeMillis();
          System.out.println("doThird()方法执行用时：" + (endTime - startTime));
          return future;
      }
  ```

  



## **属性配置优先级**

Dubbo 配置文件中各个标签属性配置的优先级总原则是：

-  方法级优先，接口级(服务级)次之，全局配置再次之。

- 如果级别一样，则消费方优先，提供方次之。

另外，还有两个标签需要说明一下：

- <dubbo:consumer/>设置在消费者端，用于设置消费者端的默认配置，即消费者端的全局设置。当然也可以设置在提供者端。但是以消费者优先级高

-  <dubbo:provider/>设置在提供者端，用于设置提供者端的默认配置，即提供者端的默认配置。当然也可以设置在消费者端。但是以消费者优先级高

**配置建议**

**provider** **上配置合理的** **provider** **端属性**

**在** **provider** **上尽量多配置** **consumer** **端属性**

- 因为provider更清楚自己的性能和服务