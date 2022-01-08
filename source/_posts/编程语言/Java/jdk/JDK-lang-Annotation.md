---
title: JDK-lang-Annotation
date: 2021-04-14 12:50:22
tags: [JDK, annotation]
---



Java 注解(Annotation)又称 Java 标注，是 JDK5.0 引入的一种注释机制。

## 使用场景

- 编译格式检查
- 反射中解析
- 生成帮助文档
- 跟踪代码依赖

### 内置注解

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufMhhxMFuSA30XB7aViccfX0Z2ngxxej4ImcDQ1VYaMh9ibaEzw6bv3csI5Lo1A7MjBbK6HY9vm3zkw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

> 补充：@SuppressWarnings 的三种方法：

1. @SuppressWarnings(“unchecked”) [^ 抑制单类型的警告]
2. @SuppressWarnings(“unchecked”,“rawtypes”) [^ 抑制多类型的警告]
3. @SuppressWarnings(“all”) [^ 抑制所有类型的警告]

### 元注解

![Image](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufMhhxMFuSA30XB7aViccfX02icFl2HhcFc4aoyhylIf0E5CtZ0hL0JSN9YLrn6GQuqWA6YeZqtj44Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Inherited 补充：

- **子类**会继承**父类**使用的注解中被@Inherited修饰的注解
- **接口继承关系**中，**子接口**不会继承**父接口**中的任何注解，不管**父接口**中使用的注解有没 被@Inherited修饰
- **子类**实现**父接口**时不会继承任何**父接口**中定义的注解

## 自定义注解



<img src="https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufMhhxMFuSA30XB7aViccfX0sVhm5T7wk0fS8pdItdfmbgvHMjcGCEBsrZAc3Ivl4tHDuZgWIMpJ9Q/640" alt="Image" style="zoom:50%;" />

### **ElementType(注解的用途类型)**

例如，若一个 Annotation 对象是 METHOD 类型，则该 Annotation 只能用来修饰方法。

```java
package java.lang.annotation;
public enum ElementType {

  TYPE,    /* 类、接口(包括注释类型)或枚举声明 */

  FIELD,     /* 字段声明(包括枚举常量) */

  METHOD,    /* 方法声明 */

  PARAMETER,   /* 参数声明 */

  CONSTRUCTOR,  /* 构造方法声明 */ 

  LOCAL_VARIABLE,  /* 局部变量声明 */ 

  ANNOTATION_TYPE,  /* 注释类型声明 */

  PACKAGE     /* 包声明 */

}
```

### **RetentionPolicy(注解作用域策略)**

```java
package java.lang.annotation;
public enum RetentionPolicy {
 SOURCE, /* Annotation信息仅存在于编译器处理期间，编译器处理完之后就没有该 Annotation信息了 */
 
 CLASS, /* 编译器将Annotation存储于类对应的.class文件中。默认行为 */
 
 RUNTIME /* 编译器将Annotation存储于class文件中，并且可由JVM读入 */
}
```

### 定义格式

```java
public @interface [自定义注解名]{

}
```

### **注意事项**

1. 定义的注解，自动继承了java.lang,annotation.Annotation接口
2. 注解中的每一个方法，实际是声明的注解配置参数

- 方法的名称就是 配置参数的名称
- 方法的返回值类型，就是配置参数的类型。只能是:基本类型/Class/String/enum

1. 可以通过default来声明参数的默认值
2. 如果只有一个参数成员，一般参数名为value
3. 注解元素必须要有值，我们定义注解元素时，经常使用空字符串、0作为默认值。

