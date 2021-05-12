---
title: springboot-starter-03-Mybatis
date: 2021-05-12 12:44:56
tags:
---

### 导入三个依赖

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
  
  <!-- druid 驱动，数据源 -->
  <dependency>
  	<groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.12</version>
  </dependency>
  ```

### 定义 Service接口

- ```java
  public class StudentService{
  	@Autowired
  	private IstudentDao dao;
  }
  ```


### 定义 **Dao** 接口

- Com.abc.dao目录

- ```java
  //Dao 接口上要添加@Mapper 注解。
  public interface IstudentDao{
  	void insertStudent(Student s);
  }
  ```

### 定义映射文件

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

### 修改主配置文件

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
