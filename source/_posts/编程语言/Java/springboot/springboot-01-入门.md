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

## Spring boot中使用线程池

- 配置类

  - 使用@Configuration和@EnableAsync这两个注解，表示这是个配置类，并且是线程池的配置类

  - ```java
    @Configuration
    @EnableAsync
    public class ExecutorConfig {
        @Bean(name = "asyncServiceExecutor")
        public Executor asyncServiceExecutor() {
            logger.info("start asyncServiceExecutor");
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            //配置核心线程数
            executor.setCorePoolSize(corePoolSize);
            //配置最大线程数
            executor.setMaxPoolSize(maxPoolSize);
            //配置队列大小
            executor.setQueueCapacity(queueCapacity);
            //配置线程池中的线程的名称前缀
            executor.setThreadNamePrefix(namePrefix);
    
            // rejection-policy：当pool已经达到max size的时候，如何处理新任务
            // CALLER_RUNS：不在新线程中执行任务，而是有调用者所在的线程来执行
            executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
            //执行初始化
            executor.initialize();
            return executor;
        }
    }
    ```

- 实现类

  - 将Service层的服务异步化，在executeAsync()方法上增加注解@Async("asyncServiceExecutor")，asyncServiceExecutor方法是前面ExecutorConfig.java中的方法名，表明executeAsync方法进入的线程池是asyncServiceExecutor方法创建的。

- 增强实现类的功能：在每次提交线程的时候都会将当前线程池的运行状况打印出来

  - Override父类的execute、submit等方法，在里面调用showThreadPoolInfo()方法

  - ```java
    private void showThreadPoolInfo(String prefix) {
            ThreadPoolExecutor threadPoolExecutor = getThreadPoolExecutor();
    
            if (null == threadPoolExecutor) {
                return;
            }
    
            logger.info("{}, {},taskCount [{}], completedTaskCount [{}], activeCount [{}], queueSize [{}]",
                    this.getThreadNamePrefix(),
                    prefix,
                    threadPoolExecutor.getTaskCount(),
                    threadPoolExecutor.getCompletedTaskCount(),
                    threadPoolExecutor.getActiveCount(),
                    threadPoolExecutor.getQueue().size());
        }
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

      