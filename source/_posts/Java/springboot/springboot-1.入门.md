---
title: springboot-1.入门
date: 2020-12-03 12:12:59
tags:
---

# Springboot基础

## 创建工程

Spring Initializr -> packaging:jar -> selected dependencies: web

Spring Initializr -> packaging:war -> selected dependencies: web

## 配置文件

Spring Boot 的主配置文件可使用 application.yml 文件。

application.properties 与 application.yml 这两个文件只能有一个。要 求文件名必须为 application。



## **Actuator** 监控器

### 基本环境搭建

#### 导入依赖

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

#### 修改配置文件

```yaml
management.server.port=9999
#指定server的基本路径
management.server.servlet.context-path=/xxx
#指定监控终端的基本路径，默认为/actuator
management.endpoints.web.base-path=/base
```

#### 测试访问health

Localhost:9999/xxx/base/health

### 添加info信息

#### 修改配置文件

在配置文件中添加如下 Info 信息，则可以通过 info 监控终端查看到。

```yaml
#自定义info信息
info.company.name=com.abc
info.company.url=http://com.abc.www

#从pom.xml文件中读取数值
info.project.groupid=@project.groupId@
info.project.artifactid=@project.artifactId@

```

#### 访问测试info

Localhost:9999/xxx/base/info

### 开放其他监控终端

默认情况下，Actuator 仅开放了 health 与 info 两个监控终端，但其还有很多终端可用，不过，需要手工开放。

#### 修改配置文件

```yaml
management.endpoints.web.exposure.include=* #开放所有终端
```

#### 测试访问

- **mappings** 终端: 
  - Localhost:9999/xxx/base/mappings
  - 通过 mappings 终端，可以看到当前工程中所有的 URI 与处理器的映射关系，及详细的 处理器方法及其映射规则。很实用。
- **beans** 终端: 
  - Localhost:9999/xxx/base/beans
  - 可以查看到当前应用中所有的对象信息。
-  **env** 终端
  - Localhost:9999/xxx/base/env
  - 查看运行环境

### 关闭其他监控终端

#### 修改配置文件

```yaml
management.endpoints.web.exposure.exclude=env,beans #关闭env,beans终端
```

#### 测试访问



# Springboot重要用法

## 自定义异常页面

对于 404、405、500 等异常状态，服务器会给出默认的异常页面，而这些异常页面一般 都是英文的，且非常不友好。我们可以通过简单的方式使用自定义异常页面，并将默认状态 码页面进行替换。

### 定义目录

在 src/main/resources 目录下再定义新的目录 public/error，必须是这个目录名称。

### 定义异常页面

在 error 目录中定义异常页面。这些异常页面的名称必须为相应的状态码，扩展名为 html。如404.html, 500.html

## 多环境选择

- 相同代码运行在不同环境
  - 开发、测试、生产 环境等。每个环境的数据库地址、服务器端口号等配置都会不同
  - 此时就需要定义出不同的配置信息，在不同的环境中选择不同的配置。
- 不同环境执行不同实现类
  - 在开发应用时，有时不同的环境，需要运行的接口的实现类也是不同的。例如，若要开发一个具有短信发送功能的应用，开发环境中要执行的 send()方法仅需调用短信模拟器即可， 而生产环境中要执行的 send()则需要调用短信运营商所提供的短信发送接口。
  - 此时就需要开发两个相关接口的实现类去实现 send()方法，然后在不同的环境中自动选 择不同的实现类去执行。

### 多配置文件实现方式

在 src/main/resources 中再定义两个配置文件，分别对应开发环境application-dev.properties与生产环境application-prod.properties。

在 Spring Boot 中多环境配置文件名需要满足 application-{profile}.properties 的格式，其 中{profile}为对应的环境标识，例如

- application-dev.properties:开发环境 
- application-test.properties:测试环境 
- application-prod.properties:生产环境

#### 相同代码运行在不同环境

至于哪个配置文件会被加载，则需要在 application.properties 文件中通过 spring.profiles.active 属性来设置，其值对应{profile}值。例如，spring.profiles.active=test 就会 加载 application-test.properties 配置文件内容。

在生产环境下，application.properties 中一般配置通用内容，并设置 spring.profiles.active 属性的值为 dev，即，直接指定要使用的配置文件为开发时的配置文件，而对于其它环境的 选择，一般是通过命令行方式去激活。配置文件 application-{profile}.properties 中则配置各个 环境的不同内容。

#### 不同环境执行不同实现类

在实现类上添加@Profile 注解，并在注解参数中指定前述配置文件中的{profile}值，用于指定该实现类所适用的环境。

- @Profile("prod")
- @Profile("dev")

### 单配置文件实现方式（略，不实用）



## 读取自定义配置

自定义配置，可以是定义在主配置文件 application.properties 中的自定义属性，也可以 是自定义配置文件中的属性。

### 读取主配置文件中的属性

在@Value 注解中通过${ }符号可以读取指定的属性值。

- 修改主配置文件

  - Student.name='xiaoming'

- 修改 **SomeHandler** 类

  - ```java
    @Value("${student.name}")
    private String name;
    ```

### 读取指定配置文件中的属性

一般情况下，[主配置文件]()中存放[系统中定义好的属性设置]()，而[自定义属性]()一般会写入[自定义的配置文件]()中。也就是说，Java 代码除了可以读取主配置文件中的属性外，还可以读取指定配置文件中的属性，可以通过@PropertySource 注解加载指定的配置文件。

> spring boot 官网给出说明，@PropertySource 注解不能加载 yml文件。所以其建议自定义配置文件就使用approperties文件。
>

- 修改property配置文件: 

  - myapp.approperties
  - 存放在 src/main/resources 目录中
  - student.name="张三"

- 修改controller类:

  - ```java
    @PropertySource(value="classpath:myapp.approperties",encoding="UTF-8")
    pulibc class testController{
      @Value("${student.name}")
    	private String name;
    }
    ```

    ​	

### 读取对象属性

- @ProertySource用于指定要读取的配置文件
- @ConfigurationProperties用于指定要读取配置文件中的对象属性，即指定要读取的配置文件属性的前辍
- @Component表示当前从配置文件读取来的对象，由Spring容器创建 
- 要保证类的属性名要与配置文件中的对象属性名相同

修改property配置文件: 

```yaml
student.id="0100"
student.name="张三"
student.class="三班"
```

修改自义配置属性类:

```java
@Component
@PropertySource("classpath:myapp.approperties")
@ConfigurationProperties("student")
public class student{
  private String id;
  private String name;
  private String class;
}
```

### 读取List< String >属性

- 导入依赖

  - 注解@ConfigurationProperties 注解需要以下依赖。

  - ```xml
    <dependency>
    	<groupId>org.springframework.boot</groupId>
    	<artifactId>spring-boot-configuration-processor</artifactId>
    	<optional>true</optional>
    </dependency>
    ```

- 修改自定义配置文件

  - ```xml
    country.cities[0]=beijing
    country.cities[1]=shanghai
    ```

- 定义配置属性类

  - ```java
    @Component
    @PropertySource("classpath:myapp.approperties")
    @ConfigurationProperties("country")
    public class Country{
      private List<String> cities;
    }
    ```



### 读取 **List< Object >**属性

- 修改自定义配置文件

  - ```yaml
    group.students[0].name = "张三"
    group.students[0].id = 01
    
    group.students[1].name = "李四"
    group.students[1].id = 02
    ```

    

- 定义配置属性类

  - ```java
    @Component
    @PropertySource("classpath:myapp.approperties")
    @ConfigurationProperties("group")
    public class Group{
      private List<Student> students;
    }
    
    //不需要添加任何注解
    public class Student{
      String name;
      String id;
    }
    ```

    

##  **Spring Boot** 下使用 **JSP** 页面

> 在 Spring Boot 下直接使用 JSP 文件，其是无法解析的，需要做专门的配置。
>
> [无法解析]()意味着访问该文件路径会下载该文件，而不是展示在页面

- 创建目录：

  - 在 src/main 下创建 webapp 目录，用于存放 jsp 文件。这就是一个普通的目录，无需执行 Mark Directory As。

- 创建 **index.jsp** :

  - [指定 **web** 资源目录](): 在 spring boot 工程中若要创建 jsp 文件，一般是需要在 src/main 下创建 webapp 目录， 然后在该目录下创建 jsp 文件。但发现没有创建 jsp 文件的选项。此时，需要打开 Project Structrue 窗口，将 webapp 目录指定为 web 资源目录，点击右下角的加号，然后才可以创建 jsp 文件。
  - 创建 **index** 页面：在webapp目录下创建index.jsp页面

- [添加tomcat内置的jsp解析器]()：

  - 添加 **jasper** 依赖, jsp 引擎是用于解析 jsp 文件的， 即将 jsp 文件解析为 Servlet 是由 jsp 引擎完成的。

  - ```xml
    <dependency>
    	<groupId>org.apache.tomcat.embed tomcat-embed-jasper</groupId>
      <artifactId>tomcat-embed-jasper</artifactId>
    </dependency>
    ```

-  [注册资源目录]():

  - 在 pom 文件中将 webapp 目录注册为资源目录。若不注册，那么target中就没有该资源，也就是说无法访问。

  - ```xml
    <build>
      <resources>
        <resource>
        	<directory>src/main/webapp</directory>
          <targetPath>META-INFO/resources</targetPath>
          <includes>
            <include>**/*.*</include>
          </includes>
        </resource>
      </resources>
    </build>
    ```

- Controller类：

  - /test/register/

  - ```
    return "jsp/welcome.jsp";
    ```

    

- 路由访问：

  - [Iocalhost:8000/index.jsp]() :访问到index.jsp页面
  - [Iocalhost:8000/test/register.jsp]() : 访问到welcome.jsp页面

## **Spring Boot** 中使用 **MyBatis**

- 导入三个依赖:

  - mybatis 与 Spring Boot 整合依赖，mysql 驱动依赖， Druid 数据源依赖。

  - ```xml
    <!--mybatis 与 spring boot 整合依赖-->
    <dependency>
    	<groupId>org.mybatis.spring.boot</groupId>
      <artifactId>mybatis-spring-boot-starter</artifactId>
    </dependency>
    
    <!--mysql 驱动-->
    <dependency>
    	<groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
    </dependency>
    
    <!-- druid 驱动 -->
    <dependency>
    	<groupId>com.alibaba</groupId>
      <artifactId>druid</artifactId>
      <version>1.1.12</version>
    </dependency>
    ```

- 定义 Service接口

  - ```java
    public class StudentService{
    	@Autowired
    	private IstudentDao dao;
    }
    ```

    

- 定义 **Dao** 接口

  - Com.abc.dao目录

  - ```java
    //Dao 接口上要添加@Mapper 注解。
    public interface IstudentDao{
    	void insertStudent(Student s);
    }
    ```

- 定义映射文件

  - Com.abc.dao目录， <!--与Dao接口在同一目录-->

  - ```xml
    <?xml version="1.0" encoding="UTF-8" ?>   
    <!DOCTYPE mapper   
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"   
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper namespace="com.abc.dao.IstudentDao">
        <!-- 这里namespace必须是UserMapper接口的路径” -->
        <insert id="insertStudent">
            insert into student(name,age) values(#{name},#{age})
        </insert>
    </mapper>
    ```

- 注册资源目录

  - 在 pom 文件中将 dao 目录注册为资源目录。

  - ```xml
     <!-- dao 目录注册为资源目录 -->
    <build>
      <resources>
        <resource>
        	<directory>src/main/java</directory>
          <includes>
            <include>**/*.xml</include>
          </includes>
        </resource>
      </resources>
    </build>
    ```

- 修改主配置文件

  - 注册映射文件

    - ```properties
      mybatis.mapper-location=classpath:com/abc/dao/.xml
      ```

  - 注册实体类别名

    - ```properties
      mybatis.type-aliases-packages=com.abc.beans
      ```

  - 注册数据源

    - ```properties
      spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
      spring.datasource.driver-class-name=com.mysql.jdbc.Driver
      spring.datasource.url=jdbc:mysql:///test
      spring.datasource.username=root
      spring.datasource.password=root
      ```

## **Spring Boot** 的事务支持

若工程直接或间接依赖于 spring-tx，则框架会自动注入 DataSourceTransactionManager事务管理器;若依赖于 spring-boot-data-jpa，则会自动注入 JpaTransactionManager。

- Spring对事务的实现：
  - 运行时异常，回滚
  - 受检查异常，提交；如果希望回滚，使用@Transactional（rollbackFor = Exception.class）

- 修改启动类

  - ```java
    @EnableTranscationManagement //开启事务
    public class Application{
    
    }
    ```

- 修改 **Service** 实现类

  - ```java
    @Transactional
    public void addStudent(Student s) throw Exception{
    	dao.insert(s);
    	//int i=0; //运行时异常，回滚
    	throw new Exception();//受检查异常，提交
    	dao.insert(s);
    }
    ```

## **Spring Boot** 对日志的控制

Spring Boot 中使用的日志技术为 logback。其与 Log4J 都出自同一人，性能要优于 Log4J， 是 Log4J 的替代者。

- 添加依赖
  
- 在 Spring Boot 中若要使用 logback，则需要具有 spring-boot-starter-logging 依赖，而该依 赖被 spring-boot-starter-web 所依赖，即不用直接导入 spring-boot-starter-logging 依赖。
  
-  添加配置文件

  - 该文件名为 logback.xml，且必须要放在 src/main/resources 类路径下。

  - ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <configuration scan="true" scanPeriod="60 seconds" debug="false" >
        <!-- 上下文名称：取项目名称 -->
        <contextName>tlogback</contextName>
        <!-- 时间戳格式 -->
        <timestamp key="myDateFormat" datePattern="yyyy-MM-dd HH:mm:ss"/>
        <!-- 属性，以${key}引用 -->
        <property name="path" value="D:/log"/>
       
        <appender name="stdout" class="ch.qos.logback.core.ConsoleAppender" >
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] [%p] %logger - %line - %msg%n</pattern>
            </encoder>
        </appender>
      
        <appender name="file" class="ch.qos.logback.core.FileAppender">
            <file>${path}/tlogback.log</file>
            <append>true</append>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] [%p] %logger - %line - %msg%n</pattern>
            </encoder>
        </appender>
    
        <!-- 为某个包，或某各类特别指定级别和appender,additivity表示是否向上级传递msg -->
        <logger name="com.abc.dao" level="debug" additivity="false">
            <appender-ref ref="stdout"/>
        </logger>
    
        <!-- 顶层的logger,为所有的类记录日志 -->
        <root>
            <appender-ref ref="stdout"/>
            <appender-ref ref="file"/>
        </root>
    
    </configuration>
    ```

## **Spring Boot** 中使用 **Redis**

> 高并发下访问 Redis，存在什么问题?存在三个问题: 
>
> - 缓存穿透: 为DB查询为null的数据预设一个值
> - 缓存雪崩: 提前规划好缓存到期时间
> - 热点缓存: 属于缓存雪崩的特例，有一个缓存到期了，大量请求访问这个缓存无效，从而大量请求数据库。双重检测锁机制

- 定义需求

  - 当前工程完成让用户在页面中输入要查询学生的 id，其首先会查看 Redis 缓存中是否存在，若存在，则直接从 Redis 中读取;若不存在，则先从 DB 中查询出来，然后再存放到 Redis 缓存中
  - 但用户也可以通过页面注册学生，一旦有新的学生注册，则需要将缓存中的学生信 息清空。根据 id 查询出的学生信息要求必须是实时性的，其适合使用注解方式的 Redis 缓存。

- 添加依赖

  - 小技巧：如redis,mybatis等依赖，[可以从父spring依赖中（ctrl + c）找到该依赖和版本号]()，然后添加

  - 在 pom 文件中添加 Spring Boot 与 Redis 整合依赖。

  - ```xml
    <!--mybatis 与 spring boot 整合依赖-->
    <dependency>
    	<groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    ```

- 修改主配置文件

  - ```properties
    #单机redis
    spring.redis.host=
    spring.redis.port=
    spring.redis.password=
    
    #集群redis
    spring.redis.sentinel.master=mymaster
    spring.redis.sentinel.nodes=sentine1:22076
    
    spring.cache.type=redis
    spring.cache.cache-names=realTimeCache
    ```

- 修改启动类

  - 添加@EnableCaching: 开启缓存

- 修改实体类 **Student**

  - 由于要将查询的实体类对象缓存到 Redis，Redis 要求实体类必须序列化。所以需要实体类实现序列化接口。

  - ```java
    public class Student implements Serializable{
    	private static final Long serialVersionUID=397293791;
    }
    ```

- 修改 **Service** 接口实现类

  - @CacheEvict(value="", allEntries="")  清楚所有缓存。业务场景，插入必须清楚所缓存

  - @Cacheable(value="", key="") 如果没有缓存，那么查数据库，并添加缓存；如果有缓存，查询指定缓存

  - ```java
    @Cacheable(value="realTimeCache", key="'student'+#id")
    public student findStudent(int id){
    	return dao.findStudent(id);
    }
    
    @CacheEvict(value="realTimeCache", allEntries="true")  
    @Transactional
    public void addStudent(Student s){
      dao.insertStudent(s);
    }
    ```

- [使用双重检查锁解决热点缓存]()

  - ```java
    public Integer getStudentsCount(){
    	BoundValueOperations<Object,Object> ops = redisTemplate.boundValuesOps("count");
      Object cnt= ops.getValue();//第一重检查
      if(cnt == null){ 
        //1nd请求来到，2nd请求，3rd请求...，因为这个锁而阻塞
        synchronized(this){
          cnt= ops.getValue();//第二重检查
          if(cnt==null){ 
            cnt = dao.findStudentsCount();
            opt.set(cnt,10,TimeUnits.Seconds);
          }
        }
      }
      
      return (Integer) cnt;
    }
    ```

  - [是否存在线程安全问题呢？]()

    - 首先，synchronized(this)中的锁必须是单例的, @Component已经保证该锁的对象在Spring容器中是单例了，所以此处没有线程安全

    - ```java
      private Integer cnt = new Integer(1);
      public Integer getStudentsCount(){
        if(cnt == null){ 
          synchronized(this){
            if(cnt==null){ 
              //以下这个new语句的底层步骤
              //1: 申请一个堆空间space
              //2: 使用对象初始数据初始化对空间space
              //3: cnt应用指向堆空间space
             	cnt = new Integer(2);
            }
          }
        }
        
        return (Integer) cnt;
      }
      ```

      - 第一个请求来到，如果2,3的步骤被编译器优化，3先执行；然后第二个请求来到，if(cnt==null)为真, 那么则返回一个没有经过2步骤的cnt对象
      - 解决方法：[cnt设置为volatile保证编译器不优化(推荐) 或者是 设置该方法为同步(略)]()

  

## Spring Boot 中使用Dubbo

  ### 步骤

  - 消费者与提供者工程均需要导入四个依赖 
    -  Dubbo 与 Spring Boot 整合依赖
    -  zkClient依赖
    -  slf4j-log4j12依赖

  - 自定义 commons 工程依赖 
  - 提供者工程
    - 将 Service 接口实现类的@Service 注解更换为阿里的注解，并添加@Component 注解
    - 在启动类上添加@EnableDubboConfiguration 与@EnableTransactionManager 注解
    - 修改配置文件:指定应用名称与注册中心地址 
  -  消费者工程
    - 将处理器中 Service 的声明上的@Autowired 注解更换为阿里的@Reference 注解 
    -  在启动类上添加@EnableDubboConfiguration 注解
    -  修改配置文件:指定应用名称与注册中心地址



### 定义 **commons** 工程

- 依赖：无，因为是纯java项目
- 定义实体类
- 定义业务接口

### 定义提供者

- 依赖：

  - 添加 **dubbo** 与 **spring boot** 整合依赖，需要从alibaba的github中找到依赖
  - 添加 **zkClient** 依赖
  - dubboCommons依赖
  - 还有其他的mysql, druid,mybatis依赖

- 定义业务接口

  - 将 Service 接口实现类的@Service 注解更换为阿里的注解，并添加@Component 注解

- 修改启动类

  - 添加@EnableDubboConfiguration 与@EnableTransactionManager 注解

- 修改配置文件

  - ```yaml
    spring:
      # 功能等价于 spring-dubbo 配置文件中的<dubbo:application/> # 该名称是由服务治理平台使用
      application:
      	name: 11-provider-springboot # 指定zk注册中心
      dubbo:
      	registry: zookeeper://zkOS:2181
      # zk 集群作注册中心
      # registry: zookeeper://zkOS1:2181?backup=zkOS2:2181,zkOS3:2181
    ```

### 定义消费者(略)

- 依赖：
  - 添加 **dubbo** 与 **spring boot** 整合依赖，需要从alibaba的github中找到依赖
  - 添加 **zkClient** 依赖
  - dubboCommons依赖

- 修改配置文件

  - ```yaml
    spring:
      # 功能等价于 spring-dubbo 配置文件中的<dubbo:application/> # 该名称是由服务治理平台使用
      application:
      	name: 11-consumer-springboot # 指定zk注册中心
      dubbo:
      	registry: zookeeper://zkOS:2181
      # zk 集群作注册中心
      # registry: zookeeper://zkOS1:2181?backup=zkOS2:2181,zkOS3:2181
    ```

    



## **Spring Boot** 下使用拦截器

> 在非 Spring Boot 工程中若要使用 SpringMVC 的拦截器，在定义好拦截器后，需要在 Spring 配置文件中对其进行注册。但 Spring Boot 工程中没有了 Spring 配置文件，那么如何使用拦截器呢?
>
> Spring Boot 对于原来在配置文件配置的内容，现在全部体现在一个类中，该类需要继承 自 WebMvcConfigurationSupport 类，并使用@Configuration 进行注解，表示该类为一个 JavaConfig 类，其充当配置文件的角色。

- 定义拦截器

  - ```java
    public class SomeInterceptor extends HandlerInterceptor{
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            String path = request.getServletPath();
            if (path.matches(Const.NO_INTERCEPTOR_PATH)) {
            	//不需要的拦截直接过
                return true;
            } else {
            	// 这写你拦截需要干的事儿，比如取缓存，SESSION，权限判断等
                System.out.println("====================================");
                return true;
            }
        }
    }
    ```

- 定义配置文件类

  - ```java
    @Configuration
    public class WebConfigurer implements WebMvcConfigurerSupport {
    	 @Override
    	 public void addInterceptors(InterceptorRegistry registry) {
            registry.addInterceptor(new SomeInterceptor())
              .addPathPatterns("/first/**");
              .excludePathPatterns("/second/**");
    	 }
    }
    ```

##  **Spring Boot** 中使用 **Servlet**

在 Spring Boot 中使用 Servlet，根据 Servlet 注册方式的不同，有两种使用方式。若使用 的是 Servlet3.0+版本，则两种方式均可使用;若使用的是 Servlet2.5 版本，则只能使用配置类方式。这里我们只记录Servlet3.0+版本。

- 创建 **Servlet**

  - @WebServlet

  - ```java
    @WebServlet(urlPatterns = "/some", asyncSupported = true)
    public class SomeServlet extends HttpServlet {
    @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            System.out.println(">>>>>>>>>>CometServlet Request<<<<<<<<<<<");
            doPost(req, resp);
    
    }
    
    ```

- 修改入口类

  - 在入口类中添加 Servlet 扫描注解。

  - ```java
    @ServletComponentScan("com.abc.servlets")
    public class Application{}
    ```

    

## **Spring Boot** 中使用 **Filter**

在 Spring Boot 中使用 Filter 与前面的使用 Servlet 相似，根据 Filter 注册方式的不同，有 两种使用方式。若使用的是 Servlet3.0+版本，则两种方式均可使用;若使用的是 Servlet2.5 版本，则只能使用配置类方式。

- 创建filter

  - @WebFilter

  - ```java
    @WebFilter(urlPatterns = "/*")
    public class SomeeFilter extends Filter {
    		@Override
        protected void doFilter(HttpServletRequest req, HttpServletResponse resp, FilterChain chain) throws ServletException, IOException {
            System.out.println(">>>>>>>>>>CometServlet Request<<<<<<<<<<<");
            chain.doFilter(req,resp);
    }
    ```

- 修改入口类

  - 在入口类中添加 Servlet 扫描注解。

  - ```java
    @ServletComponentScan("com.abc.filters")
    public class Application{}
    ```



## 路径问题基础理论

### 路径的构成

路径由两部分构成:资源路径与资源名称。[即 路径 = 资源路径 + 资源名称]()

资源路径与资源名称的分水岭为:路径中的最后一个斜杠。斜杠的前面部分称为资源路径，后面部分称为资源名称。

例如:

- 请求路径:http://localhost:8080/xxx/test/index
- 资源路径:http://localhost:8080/xxx/test
- 资源名称:index

### 路径的分类

根据是否可以唯一的定位一个资源，可以将路径划分为两类:绝对路径与相对路径。

- 绝对路径:可以唯一的定位一个资源的路径。在Web应用中，一般使用URL形式表示。

- 相对路径:仅依赖此路径无法唯一定位资源，但若为其指定一个参数路径，则可以将其转换为一个绝对路径，这样的路径称为相对路径。在 Web 应用中，一般使用 URI 形式表示。
- 转换关系: 绝对路径 = 参照路径 + 相对路径

### 绝对路径分类

根据路径作用的不同，可以将绝对路径分为:资源定义路径，与资源请求路径。

- 资源定义路径:用于表示资源位置的路径。

- 资源请求路径:客户端所发出的对指定资源的请求路径。

### 相对路径分类

根据相对路径是否以斜杠开头，可以划分为两类:

- 斜杠路径：[又可以分为两类]() <!--注意，这里是重点-->

  - 对于斜杠路径，根据其出现的位置的不同，可以划分为
    - 前台路径:
    - 出现在HTML、JS、CSS，及JSP文件的静态部分的斜杠路径。例如，<img src=""/>等等
      - 其参照路径为当前web服务器的根。
    - 后台路径:
      - 出现在Java代码、JSP文件的动态部分(Java代码块、JSP动作等)、XML、properties 等配置文件中
      - 其参照路径为当前web应用的根。
- 非杠路径：<!--建议不用-->
  - 其参照路径为当前请求路径的资源路径。



### 路径解析器

相对路径，最终都会经过路径解析器，将其转换为绝对路径，以定义或定位一个资源。 不同的相对路径，其路径解析器也是不同的。

-  前台路径:路径解析器为浏览器。

- 后台路径:路径解析器为服务器。

- 非杠路径:若非杠路径出现在前台路径位置，其路径解析器为浏览器; 若非杠路径出现在后台路径位置，其路径解析器为服务器。

### 解析规则

> 不同的路径解析器，对同一个相对路径的解析结果是不同的。所谓解析结果，指的是将 相对路径所转换为的绝对路径。
>
> 由于 绝对路径 = 参照路径 + 相对路径 ，所以不同的 解析器，会为相对路径匹配不同的参照路径。
>
> 换句话说就是，我们学习的重点是，浏览器、 服务器对于不同的相对路径所匹配的参照路径到底是谁。

- 一般规则
  
  - 斜杆路径：
    - 前台路径:其参照路径为当前web服务器的根。
    - 后台路径:其参照路径为当前web应用的根。
  - 非杠路径:其参照路径为当前请求路径的资源路径。
  
- 例子，请求路径: http://localhost:8080/xxx/test/index

  - 当前 web 服务器的根: http://localhost:8080
  - 当前web应用的根: http://localhost:8080/xxx
  - 资源路径: http://localhost:8080/xxx/test

- 特殊规则：

  - 在代码中使用HttpServletResponse的sendRedirect()方法使用斜杠路径进行重定向时， 其参照路径按照之前规则，应该是当前 web 应用的根，但实际情况是，当前 web 服务器的根。

  - 以上规则对于一次跳转是没有问题的，若跳转次数超过一次，则有可能会存在问题。

  - 如何解决呢？

    - ```java
      response.sendRedirect(request.getContextPath()+"/html/welcome.html");
      ```

      