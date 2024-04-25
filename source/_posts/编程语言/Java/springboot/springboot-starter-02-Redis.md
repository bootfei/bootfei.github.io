---
title: springboot-starter-02-redis
date: 2021-05-12 12:20:17
tags:
---



> 高并发下访问 Redis，存在什么问题?存在三个问题: 
>
> - 缓存穿透: 为DB查询为null的数据预设一个值
> - 缓存雪崩: 提前规划好缓存到期时间
> - 热点缓存: 属于缓存雪崩的特例，有一个缓存到期了，大量请求访问这个缓存无效，从而大量请求数据库。双重检测锁机制



### 添加依赖

- 小技巧：如redis,mybatis等依赖，[可以从父spring依赖中（ctrl + c）找到该依赖和版本号]()，然后添加

- 在 pom 文件中添加 Spring Boot 与 Redis 整合依赖。

- ```xml
  <!--mybatis 与 spring boot 整合依赖-->
  <dependency>
  	<groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>
  ```

### 修改主配置文件

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

### 修改启动类

- 添加@EnableCaching: 开启缓存

### 修改实体类 **Student**

- 由于要将查询的实体类对象缓存到 Redis，Redis 要求实体类必须序列化。所以需要实体类实现序列化接口Serializable。


### 修改 **Service** 接口实现类

@CacheEvict(value="", allEntries="")  清楚所有缓存。业务场景，插入必须清楚所缓存

@Cacheable(value="", key="") 如果没有缓存，那么查数据库，并添加缓存；如果有缓存，查询指定缓存

```java
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

### 解决缓存问题

#### [使用双重检查锁解决热点缓存]()

- ```java
  public Integer getStudentsCount(){
  	BoundValueOperations<Object,Object> ops = redisTemplate.boundValuesOps("count");
    Object cnt= ops.getValue();//第一重检查
    if(cnt == null){ 
      //1nd请求来到，2nd请求和3rd请求，因为这个锁而阻塞，无法查询数据库
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

#### [是否存在线程安全问题呢？]()

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
  
    - 第一个请求来到，如果2,3的步骤被编译器优化导致3先执行；然后第二个请求来到，if(cnt==null)为真, 那么则返回一个没有经过2步骤的cnt对象
    - 解决方法：[cnt设置为volatile保证编译器不优化(推荐) 或者是 设置该方法为同步(略)]()

