---
title: springboot-02-ioc和di
date: 2020-10-17 19:52:16
categories: [java,spring]
tags:
---



# 核心概念

1. 自动装配
2. 起步依赖
3. 内置Tomcat
4. 可以以jar方式去部署启动

## 容器

- 基础容器BeanFactory：第一次被调用以后才产生
- 高级容器ApplicationContext：在程序启动的时候就生成所有单例bean实例

## 循环依赖

 * A-->B	B-->A形成闭环
      * 构造方法的循环依赖（无法解决的）
      * setter方法的循环依赖（Spring使用三级缓存技术解决）





# IOC模块

> 需求：实现用户查询功能
>
>    * 业务层UserService
>      		* UserServiceImpl
>      	*	持久层
>      		* UserDao
>      		* UserDaoImpl
>      	* PO层
>      		* User

## 实现1: 不用IOC

```java
//业务代码块
{
    //与业务无关，完全可以交给Spring创建
    UserService userService = new UserServiceImpl();
    UserDao dao = new UserDaoImpl();
    BasicDataSource dataSource = new User;
    dataSource.setDriverName("mysql.driver.class");
    dataSource.setUser("root");
        
    userDao.setDataSource(dataSource);
	userservice.setUserDao(userDao);
    
    User user= new User();
    user.setUsername("王五");
    List<User> users =  userServie.queryUsers(user);
}
```



## 实现2：面向过程，交给IOC创建

1. 采取Map集合来存储单例Bean实例, 通过BeanName来获取实例
2. 需要通过xml来配置bean信息(参考Spring的配置)
3. 将XML中配置的每个Bean的信息封装到一个Java对象里面保存（BeanDefinition），最终将bean的名称和BeanDefinition对象封装到Map集合中

```java
//业务代码块
{
   	UserService userService = getBean();
   	
    userDao.setDataSource(dataSource);
	userservice.setUserDao(userDao);
    
    User user= new User();
    user.setUsername("王五");
    List<User> users =  userServie.queryUsers(user);
}
```



### 一级缓存Bean实例和BeanDefinitions存储Bean元信息

```java
//k:BeanName v:Bean实例对象
private Map<String,Object> singletonObjects = new HashMap<String, Object>();

//k:BeanName v:BeanDefinition对象
private Map<String,BeanDefinition> beanDefinitions = new HashMap<String, Object>();

//解析xml
@Before
public void init(){
		// 饿汉式
		// singletonObjects.put(key, value);
		// getBean();

		// XML解析，将BeanDefinition注册到beanDefinitions集合中
		String location = "beans.xml";
		// 获取流对象
		InputStream inputStream = this.getClass().getClassLoader().getResourceAsStream(location);
		// 创建文档对象
		Document document = createDocument(inputStream);

		// 按照spring定义的标签语义去解析Document文档
		registerBeanDefinitions(document.getRootElement());
}

//获取Bean
private Object getBean(String beanName){
        //先从一级缓存获取单例Bean的实例
        Object singletonObject= singletonObjects.get(beanName);	

        if(singletonObject !=null) return singletonObject;
        //懒汉式：等用户getBean时，并且singletonObjects没有该实例时，才创建该实例
        //当缓存中没有找到该Bean时，则需要创建Bean，然后将该Bean放入一级缓存
        //要创建Bean，必须先获取Bean信息（从xml文件获取）

       // 根据beanName去beanDefinitions获取对应的Bean信息
        BeanDefinition beanDefinition = this.beanDefinitions.get(beanName);
        if (beanDefinition == null || beanDefinition.getClazzName() == null) {
            return null;
        }

        // 根据Bean的信息，来判断该bean是单例bean还是多例（原型）bean
        if (beanDefinition.isSingleton()) {
            // 根据Bean的信息去创建Bean的对象
            singletonObject = createBean(beanDefinition);
            // 将Bean的对象，存入到singletonObjects
            this.singletonObjects.put(beanName, singletonObject);
        } else if (beanDefinition.isPrototype()) {
            // 根据Bean的信息去创建Bean的对象
            singletonObject = createBean(beanDefinition);
        } else {
            // TODO 。。。
        }

        return singletonObject;
}


```



### BeanDefinition.class存储Bean信息

```java
//id，class，scope，init-method
//List<PropertyValue对象> 有name，Object value

public class BeanDefinition{
    private String id;
    private String name; //id和name2选1
    private String class;
}

public class PropertyValue{
    
}

```

### createBean(BeanDefinition)创建单例Bean

```
private Object createBean(BeanDefinition beanDefinition) {
		// TODO 第一步：new对象（初始化）
		// TODO 第二步：依赖注入
		// TODO 第三步：初始化，就是调用initMethod指定的初始化方法，或者实现了InitializingBean接口的afterPropertiesSet方法----
		return null;
}
```





## 实现3:

