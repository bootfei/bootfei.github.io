---
title: JDK-lang-Reflect
date: 2021-04-14 12:50:22
tags: [JDK,reflect]
---

## 反射

JAVA反射机制是在运行状态中，获取任意一个类的结构 , 创建对象 , 得到方法，执行方法 , 属性；

这种在运行状态动态获取信息以及动态调用对象方法的功能被称为java语言的反射机制。

### 类加载器

Java Classloader 是JRE的一部分，负责动态加载Java类到Java虚拟机的内存空间中。

**BootstrapClassLoader:**嵌在JVM内核中的加载器，该加载器是用C++语言写的，主要负载加载JAVA_HOME/lib下的类库，引导启动类加载器无法被应用程序直接使用。

**ExtensionClassLoader(扩展类加载器):**ExtensionClassLoader是用JAVA编写，且它的父类加载器是Bootstrap。是由`sun.misc.Launcher$ExtClassLoader`实现的，主要加载JAVA_HOME/lib/ext目录中的类 库。它的父加载器是BootstrapClassLoader

**App ClassLoader(应用类加载器):**App ClassLoader是应用程序类加载器，负责加载应用程序classpath目录下的所有jar和class文件。它的父加载器为Ext ClassLoader

**双亲委派模型:**（略）

类通常是**懒加载**，即第一次使用该类时才加载。由于有了类加载器，Java运行时系统不需要知道文件与 文件系统。

### 加载配置文件

1. 给项目添加resource root目录
2. 通过类加载器加载资源文件

- 默认加载的是src路径下的文件，但是当项目存在resource root目录时，就变为了加载 resource root下的文件了。

### 反射获取Class

要想了解一个类,必须先要获取到该类的字节码文件对象。每一个字节码文件，在内存中都存在一个对应的Class类型的对象。

```
Class.forName(com.test.Student.class)：包名+类名，得到student类
Student.getClass()：得到Student类
```

- 如果类在内存中不存在, 则会加载到内存。

- 如果类已经在内存中存在, 不会重复加载, 而是重复利用

**特殊的类对象**

> 基本数据类型的类对象：基本数据类型.class
>
> 包装类.type
>
> 基本数据类型包装类对象:包装类.class

### 反射获取 Constructor

```
构造方法如下:
Person(String name,int age)

1.通过指定的参数类型,得到这个public构造方法:
Constructor c = p.getClass().getConstructor(String.class,int.class);

2. 获取所有public构造方法
Constructor[] c = p.getClass().getConstructors();

3.获取所有权限的单个构造方法
getDeclaredConstructor(参数类型的class对象数组)

4.获取所有权限的构造方法
Constructor[] c = getDeclaredConstructors();
```

### 反射通过Constructor 创建对象

```
Object class = newInstance(Object... para);
Student std = (Student)class;
```

### 反射获取 Method

```
1.通过class对象 获取一个类的方法
getMethod(String methodName , class.. clss)

2.得到一个类的所有方法 (public修饰的)
getMethods();

3.根据参数列表的类型和方法名, 得到一个方法(除继承以外所有的:包含私有, 共有, 保护, 默认)
getDeclaredMethod(String methodName , class.. clss)

4.得到一个类的所有方法 (除继承以外所有的:包含私有, 共有, 保护, 默认)
getDeclaredMethods();
```

**Method 执行方法**

```
1.调用方法,参数1:要调用方法的对象;参数2:要传递的参数列表
method.invoke(Object o,Object... para) :
```

 

### 反射获取 Field

```
1.根据属性的名称，获取一个类的属性
clz.getDeclaredField(String filedName)

2.获取所有属性
clz.getDeclaredFields()

3.根据属性的名称，获取一个类的public属性
getField(String filedName)

4.获取所有属性 (public)
getFields()
```

**Field 属性的对象类型**

```
1.参数: 要获取属性的对象 获取指定对象的此属性值
field.get(Object o);
   
2.参数1：要设置属性值的对象；参数2：要设置的值设置指定对象的属性的值
field.set(Object o , Object value);
   
3.获取属性的名称 
field.getName();
   
4.如果flag为true 则表示忽略访问权限检查 !(可以访问任何权限的属性)
setAccessible(boolean flag);
```

   

### 通过反射获取注解信息

```java
//获取类/属性/方法的全部注解对象
Annotation[] annotations01 = Class/Field/Method.getAnnotations();

for (Annotation annotation : annotations01) {
    System.out.println(annotation);
}

//根据类型获取类/属性/方法的注解对象
注解类型 对象名 = (注解类型) c.getAnnotation(注解类型.class);
```

### Introspector获取JavaBean

基于反射 , java所提供的一套应用到JavaBean的API

Bean类的定义：

- 一个定义在包中的类 
- 拥有无参构造器
- 所有属性私有
- 所有属性提供get/set方法
- 实现了序列化接口

Java提供了一套java.beans包的api , 对于反射的操作, 进行了封装

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufMhhxMFuSA30XB7aViccfX0PEZpoWVlXU9Qf2EibCia3dlsHN72X9EGdkDMMExWicL1WibZ5qVtDUs9Qw/640)

```
##获取Bean类信息
BeanInfo getBeanInfo(Class cls)

##获取bean类的get/set方法数组
MethodDescriptor[] getPropertyDescriptors():

##MethodDescriptor
Method getReadMethod():获取一个get方法
Method getWriteMethod():获取一个set方法。有可能返回null 注意 需要加判断
```

