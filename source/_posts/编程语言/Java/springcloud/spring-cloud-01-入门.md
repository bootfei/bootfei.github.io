---
title: 'spring-cloud-1.入门'
date: 2020-12-12 15:40:07
tags:
---

[推荐教程网站](https://book.itheima.net/course/1265899443273850881/1275268490151862274/1275269858560319489)

# spring cloud 与spring boot关系

- Spring Cloud 与 Spring Boot 是什么关系呢?Spring Boot 为 Spring Cloud 提供了代码实现 环境，使用 Spring Boot 将其它组件有机融合到了 Spring Cloud 的体系架构中了。所以说，Spring Cloud 是基于 Spring Boot 的、微服务系统架构的一站式解决方案。
- <img src="/Users/qifei/Library/Application Support/typora-user-images/image-20201212154317497.png" alt="image-20201212154317497" style="zoom: 25%;" />



# 第一个服务提供者**/**消费者项目

本例实现了消费者对提供者的调用，但并未使用到Spring Cloud，但其为后续Spring Cloud 的运行测试环境。使用 MySQL 数据库，使用 Spring Data JPA 作为持久层技术。

## 创建provider module

- Maven依赖


```xml
<groupId>com.abc</groupId>
<artifactId>01-provider-8081</artifactId>
	
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
        <version>1.1.10</version>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

- 实体类


```java
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import lombok.Data;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Data
@Entity //自动建表，例如@Entity(name = "t_depart")，如果不写name，那么默认使用类名称;
@JsonIgnoreProperties({"hibernateLazyInitializer","handler","fieldHandler"}) 
//服务端和客户端使用json, 所以需要java与json转换，由HttpMessagerConverter完成，具体交给Jackson完成(java与json对象的转换)。
//JPA默认hibernate, 而hibernate默认对于对象的查询基于延迟加载
//比如, Depart d = service.getDepart(1); 并不会真正的执行底层的dao.select()，而是将结果封装到Depart d中；
// String name = d.getName();此时才真正的查询；
//但是 Depart d要被Jackson转换，但是是空的数据，所以hibernate必须关闭延迟加载，即忽略
public class Depart {
    @Id //表示当前属性为自动建表的主键
    @GeneratedValue(strategy = GenerationType.IDENTITY) //表示递增
    private Integer id;
    private String name;
}
```

- 定义Repository接口


```java
import com.abc.entity.Depart;
import org.springframework.data.jpa.repository.JpaRepository;

public interface DepartRepository extends JpaRepository<Depart,Integer> {
}
```

- 定义service接口以及service实现


```java
@Service
public class DepartServiceImpl implements DepartService {
    @Autowired
    private DepartRepository repository;

    @Override
    public boolean save(Depart depart) {

        Depart obj = repository.save(depart);
        if (obj == null)
            return false;
        return true;

    }

    @Override
    public Depart getDepartById(Integer id) {
        try {
            Depart depart = repository.getOne(id);
            return depart;
        } catch (Exception e) {
            return null;
        }
    }

}
```

- 定义处理器


```java
@RestController
@RequestMapping("/provider/depart")
public class ProviderController {
    @Autowired
    private DepartService service;

    @PostMapping("/save")
    public boolean saveHandle(@RequestBody Depart depart) {
        return service.save(depart);
    }

    @GetMapping("/get/{id}")
    public Depart getHandle(@PathVariable(value = "id")int id){
        return service.getDepartById(id);
    }
}
```

- 配置yml文件


```yml
server:
  port: 8081

spring:
  jpa:
    generate-ddl: true ##自动生成表
    show-sql: true ##显示sql语句
    hibernate:
      ddl-auto: none

  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
    url: jdbc:mysql://localhost/test?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
    username: root
    password: root
```

## 创建consumer module

- Maven依赖


```xml
    <groupId>com.abc</groupId>
    <artifactId>01-consumer-8080</artifactId>
     <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
```

- 修改 **JavaConfig** 类
  - 配置RestTemplate

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class DepartCodeConfig {
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

- 实体类: 纯java


```java
import lombok.Data;
@Data
public class Depart {
    private Integer id;
    private String name;
}
```

- controller
  - 通过Http直接请求提供者的接口

```java
@RestController
@RequestMapping("/consumer/depart")
public class ConsumerController {
    @Autowired
    private RestTemplate restTemplate;

    private final String  SERVICE_PROVIDER="http://localhost:8081";

    @RequestMapping("/save")
    public boolean saveHandler(@RequestBody Depart depart){
        String url = SERVICE_PROVIDER + "/provider/depart/save";
        return restTemplate.postForObject(url, depart,Boolean.class);
    }

    @GetMapping("/get/{id}")
    public Depart getHandler(@PathVariable(value = "id") int id){
        String url = SERVICE_PROVIDER + "/provider/depart/get/"+id;
        return restTemplate.getForObject(url,Depart.class);
    }
}
```



