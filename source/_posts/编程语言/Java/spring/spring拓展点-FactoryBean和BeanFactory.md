---
title: springboot-FactoryBean和BeanFactory
date: 2021-04-12 09:13:20
tags:
---



FactoryBean: 在BeanFactory提供Bean之前，需要事先定义这个bean由哪些组件组成

## FactoryBean应用

BeanFactory是生产bean的工厂并且通过getBean接口进行获取。但是在通过getBean获取bean之前，需要事先定义这个bean由哪些组件组成。定义的方式有很多，可以通过xml进行定义，或者在代码中通过注解（@Bean、@Service）进行定义。

这样FactoryBean的就有了其意义，它可以定义出一种类型的Bean,并且在创建的时候再去实现其具体的功能。里面有三个方法。

- `getObject` 获取bean方法，在此方法中，我们可以自己定义一个对象，然后自行修改其创建过程。通过这个方法，我们可以在mapper创建的时候再实现其具体的功能。
- `getObjectType` 获取这类的类型。
- `isSingleton` 是否单例。

```
public interface FactoryBean<T> {
    String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

    @Nullable
    T getObject() throws Exception;

    @Nullable
    Class<?> getObjectType();

    default boolean isSingleton() {
        return true;
    }
}
```

## FactoryBean源码

这里带领大家了解下Mybatis的MapperFactoryBean，这个是生成Mapper的FactoryBean。

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudngSXpAkDiasH1qK8ojKKCjb1k1G4PNRZ0nhyIuzyuoetyQabviaej3Phw7lmjtzaCrB5m1pxIHBAw/640)

通过上图的流程即可发现。每一个mapper是通过MapperFactoryBean的getObject方法进行创建，最后生成一个代理类。在代理类中对Mapper对应的注解信息进行解析。

例如spring cloud的feign组件，里面肯定也会看到FactoryBean的身影。

对于mybatis和feign，可以很轻松的发现其共同点：

- 存在一种类型的bean。mybatis是Mapper,feign是FeignClient。
- 这种bean功能单一。mapper只跟数据库做交互。FeignClient只是接口调用。
- quartz框架。里面也有JobDetailFactoryBean
- Redis中有RedisClientFactoryBean。
- security框架的UserDetailsManagerResourceFactoryBean。
- shiro框架的ShiroFilterFactoryBean  <!--我正在研究的框架-->

其实他们都是有一个共同的特点，就是生产的bean是一种类型，在[创建的过程中]()在实现其功能