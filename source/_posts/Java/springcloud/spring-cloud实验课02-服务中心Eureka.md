---
title: spring cloud实验课02 服务中心Eureka
date: 2020-12-12 17:08:01
tags:
---



Ref: 

1. 官网介绍
   https://github.com/Netflix/eureka/wiki



1. 创建Eureka Server module，以及搭建Eureka Server集群

   1. Maven依赖

      ```xml
      //spring cloud依赖
      <properties>
      		<java.version>1.8</java.version>
      		<spring-cloud.version>Hoxton.SR1</spring-cloud.version>
      	</properties>
      <dependencyManagement>
      		<dependencies>
      			<dependency>
      				<groupId>org.springframework.cloud</groupId>
      				<artifactId>spring-cloud-dependencies</artifactId>
      				<version>${spring-cloud.version}</version>
      				<type>pom</type>
      				<scope>import</scope>
      			</dependency>
      		</dependencies>
      </dependencyManagement>
      
      //eureka server依赖
      <dependencies>
        <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
      </dependencies>
      ```

   2. 配置文件

      ```yml
      server:
      	port: 8000
      	
      eureka:
        client:
          serviceUrl:
            defaultZone: http://localhost:8000/eureka/
            #若搭建Eureak Server集群，则改为以下配置
            defaultZone: http://localhost:8000/eureka/,http://localhost:8100/eureka/
            
        server:
      	  #开启自我保护机制
        	enable-self-preservation: true
        	#指定开启阈值
        	renewal-percent-threshold: 0.75
      ```

   3. 定义spring boot启动类

      ```java
      @SpringBootApplication
      @EnableEurekaServer //非常关键！！！
      public class Application {
      
          public static void main(String[] args) {
              new SpringApplicationBuilder(Application.class).web(true).run(args);
          }
      
      }
      ```

2. 创建provider module
   注意：与实验一大致相同，只记录不同部分

   1. Maven依赖

      ```xml
      //spring cloud依赖
      <properties>
      		<java.version>1.8</java.version>
      		<spring-cloud.version>Hoxton.SR1</spring-cloud.version>
      	</properties>
      <dependencyManagement>
      		<dependencies>
      			<dependency>
      				<groupId>org.springframework.cloud</groupId>
      				<artifactId>spring-cloud-dependencies</artifactId>
      				<version>${spring-cloud.version}</version>
      				<type>pom</type>
      				<scope>import</scope>
      			</dependency>
      		</dependencies>
      </dependencyManagement>
      
      
      <dependencies>
        //eureka client依赖
        <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
       
      
        //actuator依赖
        <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
      </dependencies>
      ```

   2.  配置文件

      ```yaml
      eureka:
        client:
          serviceUrl:
            defaultZone: http://localhost:8000/eureka/
            #若使用Eureka集群，则改为以下配置
            defaultZone: http://localhost:8000/eureka/,http://localhost:8100/eureka/
          instance:
          	instance-id: abc-msc-provider-8081
      
      spring:
      	application:
      		name: abcmsc-provider-depart
      ```

   3. 定义spring boot启动类
      从官网介绍看到，启动类不需要做修改，eureka已经做了

      <img src="/Users/qifei/Library/Application Support/typora-user-images/image-20201212173428818.png" alt="image-20201212173428818" style="zoom:25%;" />

      

   4. Controller

      ```java
      @Autowired
      private DiscoveryClient client; //服务发现
      ```

      





