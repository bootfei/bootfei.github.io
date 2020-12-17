---
title: 'spring cloud实验课01 spring cloud入门'
date: 2020-12-12 15:40:07
tags:
---



1. spring cloud 与spring boot关系
   <img src="/Users/qifei/Library/Application Support/typora-user-images/image-20201212154317497.png" alt="image-20201212154317497" style="zoom: 25%;" />

   
   

2. 实验准备

   1. 创建provider module

      1. Maven依赖

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

      2. 实体类

         ```java
         import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
         import lombok.Data;
         import javax.persistence.Entity;
         import javax.persistence.GeneratedValue;
         import javax.persistence.GenerationType;
         import javax.persistence.Id;
         
         @Data
         @Entity
         @JsonIgnoreProperties({"hibernateLazyInitializer","handler","fieldHandler"})
         public class Depart {
             @Id
             @GeneratedValue(strategy = GenerationType.IDENTITY)
             private Integer id;
             private String name;
         }
         ```

      3. 定义Repository接口

         ```java
         import com.abc.entity.Depart;
         import org.springframework.data.jpa.repository.JpaRepository;
         
         public interface DepartRepository extends JpaRepository<Depart,Integer> {
         }
         ```

      4. 定义service接口以及service实现

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

      5. 定义处理器

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

      6. 配置yml文件

         ```yml
         server:
           port: 8081
         
         spring:
           jpa:
             generate-ddl: true
             show-sql: true
             hibernate:
               ddl-auto: none
         
           datasource:
             driver-class-name: com.mysql.jdbc.Driver
             type: com.alibaba.druid.pool.DruidDataSource
             url: jdbc:mysql://localhost/test?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
             username: root
             password: root
         ```

   2. 创建consumer module

      1. Maven依赖

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

      2. config配置RestTemplate

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

      3. 实体类: 纯java

         ```java
         import lombok.Data;
         @Data
         public class Depart {
             private Integer id;
             private String name;
         }
         ```

      4. controller

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

         

