---
title: springboot-FactoryBean和BeanFactory
date: 2021-04-12 09:13:20
tags:
---

## 前言

常说spring的核心是ioc，ioc的核心是BeanFactory。然而在spring中还有一个很容易让人混淆的词FactoryBean。本文通过一些mybatis源码来讲述其区别，请大家参考。

## 一、为什么会有FactoryBean？

BeanFactory是生产bean的工厂。在此工厂中，我们可以生产出我们想要的bean，并且通过getBean接口进行获取。

但是在通过getBean获取bean之前，我们需要事先定义这个bean涨什么样子，或者说它由哪些组件组成。定义的方式有很多，可以通过xml进行定义，或者在代码中通过注解（@Bean、@Service）进行定义。

就好比一个Controller，在最原始的xml配置bean的时候，我们需要定义它是由哪些service组成，然后一点点的配置好。xml要与Controller的service一一对应起来。

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudngSXpAkDiasH1qK8ojKKCjNLksdkxibGFpjw28Sl9aRkr1BUhHukCibTrTf5IB9Jm3eGLtPQ227m5A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

这种方式的弊端是所有的bean都需要事先定义好，但是有时候，有的一些bean，我们只知道它大概的样子，但是无法事先定义出其具体的功能。

就好比，我们知道它是一只鸟，但是不知道是什么种类的鸟，只有在代码执行的时候，我才知道是什么种类的鸟。

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

## 二、通过源码深入学习FactoryBean

这里带领大家了解下Mybatis的MapperFactoryBean，这个是生成Mapper的FactoryBean。

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbudngSXpAkDiasH1qK8ojKKCjb1k1G4PNRZ0nhyIuzyuoetyQabviaej3Phw7lmjtzaCrB5m1pxIHBAw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

大家可以自行打开源码查看，通过上图的流程即可发现。每一个mapper是通过MapperFactoryBean的getObject方法进行创建，最后生成一个代理类。在代理类中对Mapper对应的注解信息进行解析。

相信跟一下mybatis的源码之后，对FactoryBean会有更加深入的理解。虽然在开发时用FactoryBean的机会并不多，但是源码中会经常遇到，例如spring cloud的feign组件，里面肯定也会看到FactoryBean的身影。

对于mybatis和feign，可以很轻松的发现其共同点：

- 存在一种类型的bean。mybatis是Mapper,feign是FeignClient。
- 这种bean功能单一。mapper只跟数据库做交互。FeignClient只是接口调用。
- quartz框架。里面也有JobDetailFactoryBean
- Redis中有RedisClientFactoryBean。
- security框架的UserDetailsManagerResourceFactoryBean。

其实他们都是有一个共同的特点，就是生产的bean是一种类型，在[创建的过程中]()在实现其功能