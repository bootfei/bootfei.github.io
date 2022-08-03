---
title: 'chapter7:虚拟机类加载机制'
date: 2020-12-23 18:42:39
tags: [java]
---

ref: https://mp.weixin.qq.com/s/GbWAGFZqILdFJdGVLNpvRw



![图片](https://mmbiz.qpic.cn/mmbiz_png/1TAcib2scKMT9n0lSVtWic7tmb0WgrrbBkicnpaWJNscP6888C0Bqd6zSsWTBicbWx5aW7KY9EkXGl63hAtRBlARfQ/640)

# 概述

代码编译的结果从本地机器码转变为字节码，是存储格式发展的一小步



**在Class文件中描述的各种信息，最终都需要加载到虚拟机中之后才能被运行和使用。**

虚拟机把描述类的数据从**Class文件加载到内存**，并**对数据进行校验**、**转换解析**和**初始化**，最终形成可以被虚拟机**直接使用的Java类型**，这就是虚拟机的`类加载机制`。

与那些在编译时需要进行连接工作的语言不同，在Java语言里面，类型的`加载`和`连接`过程都是在`程序运行期间`完成的，这样会在类加载时稍微增加一些性能开销，但是却能为Java应用程序提供高度的灵活性，

Java中天生可以动态扩展的语言特性就是依赖`运行期动态加载`和`动态连接`这个特点实现的。例如，如果编写一个使用接口的应用程序，可以等到运行时再指定其实际的实现。

<!--“Class文件”并非指Class必须是存在于具体磁盘中的某个文件，这里说的Class文件指的是一串二进制的字节流，无论以何种形式存在都可以。-->



# 类加载的时机

类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括了：

> 加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、
>
> 初始化（Initialization）、使用（Using）和卸载（Unloading）七个阶段。

![类加载机制](https://zoctan.github.io/2018/07/25/Java/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.png)

`加载`、`验证`、`准备`、`初始化`和`卸载`这五个阶段的顺序是确定的，`类的加载`过程必须按照这种顺序按部就班地开始，

## 加载开始的时机

对于**`加载`**阶段开始的时间，虚拟机规范中并没有进行强制约束，这点可以交给虚拟机的具体实现来自由把握。

## 解析开始的时机

`解析`在某些情况下可以在`初始化`阶段之后再开始，这是为了支持Java语言的运行时绑定（也称为动态绑定或晚期绑定）。<!--请注意这里写的是按部就班地“开始”，而不是按部就班地“进行”或“完成”，因为这些阶段通常都是互相交叉地混合式进行的，通常会在一个阶段执行的过程中调用或激活另外一个阶段。-->

## 初始化开始的时机

对于**`初始化`**阶段，虚拟机规范则是严格规定了有且只有4种情况必须立即对类进行`初始化` 

> `加载`、`验证`、`准备`自然需要在此之前开始

1. 遇到`new`、`getstatic`、`putstatic`、`invokestatic`这4条字节码指令时，如果类没有进行过`初始化`，则需要先触发其`初始化`。生成这4条指令的最常见的Java代码场景是：
   - 使用`new`关键字实例化对象的时候
   - 读取或设置一个类的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候
   - 调用一个类的静态方法的时候
2. 使用java.lang.reflect包的方法对类进行[反射]()调用的时候，如果类没有进行过初始化，则需要先触发其初始化。
3. 当初始化一个类的时候，[如果发现其父类还没有进行过初始化]()，则需要先触发其父类的初始化。
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含[main()方法的那个类]()），虚拟机会先初始化这个主类。

对于这四种会触发类进行初始化的场景，虚拟机规范中使用了一个很强烈的限定语：“**有且只有**”，这四种场景中的行为称为对一个类进行`主动引用`。

除此之外所有引用类的方式，都不会触发初始化，称为`被动引用`。

下面举三个例子来说明被动引用，分别见代码清单7-1、代码清单7-2和代码清单7-3。

代码清单7-1　被动引用的例子之一

------

```java
package org.fenixsoft.classloading;
/**
 * 被动使用类字段演示一：
 * 通过子类引用父类的静态字段，不会导致子类初始化
 **/
public class SuperClass {
  static {
  　　 System.out.println("SuperClass init!");
  }
  public static int value = 123;
}
public class SubClass extends SuperClass {
  static {
    System.out.println("SubClass init!");
  }
}
/**
 * 非主动使用类字段演示
 **/
public class NotInitialization {
  public static void main(String[] args) {
    System.out.println(SubClass.value);
  }
}
```

------

上述代码运行之后，只会输出“SuperClass init！”，而不会输出“SubClass init！”。对于静态字段，只有直接定义这个字段的类才会被`初始化`，因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的`初始化`。至于是否要触发子类的`加载`和`验证`，在虚拟机规范中并未明确规定。

代码清单7-2　被动引用的例子之二

------

```java
package org.fenixsoft.classloading;
/**
 * 被动使用类字段演示二：
 * 通过数组定义来引用类，不会触发此类的初始化
 **/
public class NotInitialization {
  public static void main(String[] args) {
    SuperClass[] sca = new SuperClass[10];
  }
}
```

------

这段代码为了节省版面，复用了代码清单7-1中的SuperClass，运行之后发现没有输出“SuperClass init！"，说明并没有触发类org.fenixsoft.classloading.SuperClass的初始化阶段。

代码清单7-3　被动引用的例子之三

------

```java
package org.fenixsoft.classloading;
/**
 * 被动使用类字段演示三：
 * 常量在编译阶段会存入调用类的常量池中，本质上没有直接引用到定义常量的类，因此不会触发定义常量的类
的初始化。
 **/
public class ConstClass {
  static {
    System.out.println("ConstClass init!");
  }
  public static final String HELLOWORLD = "hello world";
}
/**
 * 非主动使用类字段演示
 **/
public class NotInitialization {
  public static void main(String[] args) {
    System.out.println(ConstClass.HELLOWORLD);
  }
}
```

------

上述代码运行之后，也没有输出“ConstClass init！”，这是因为虽然在Java源码中引用了ConstClass类中的常量HELLOWORLD，但是在编译阶段将此常量的值“hello world”存储到了NotInitialization类的常量池中，对常量ConstClass.HELLOWORLD的引用实际都被转化为NotInitialization类对自身常量池的引用了。

> 也就是说**实际上NotInitialization的Class文件之中并没有ConstClass类的符号引用入口**，这两个类在编译成Class之后就不存在任何联系了。

接口的加载过程与类加载过程稍有一些不同，针对接口需要做一些特殊说明：接口也有初始化过程，这点与类是一致的，上面的代码都是用静态语句块“static{}”来输出初始化信息的，而接口中不能使用“static{}”语句块，但编译器仍然会为接口生成“<clinit>()”类构造器 [[2\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00045.html#ch2-back) ，用于初始化接口中所定义的成员变量。

接口与类真正有所区别的是前面讲述的四种“**有且仅有**”需要开始初始化场景中的第三种：

- 当一个类在`初始化`时，要求其父类全部都已经`初始化`过了，但是一个接口在`初始化`时，并不要求其父接口全部都完成了`初始化`，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化。



[[2\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00045.html#ch2) 关于类构造器<clinit> 和方法构造器<init> 的生成过程和作用，可参见第10章的相关内容。

# 类加载的过程

接下来我们详细讲解一下类加载的全过程，也就是加载、验证、准备、解析和初始化这五个阶段的过程。

## 加载

**`加载`**（Loading）阶段是“**`类加载`**”（Class Loading）过程的一个阶段

在**`加载`**阶段，虚拟机需要完成以下三件事情：

1）通过一个类的全限定名来获取定义此类的二进制字节流。

2）将这个字节流所代表的静态存储结构转化为[方法区的运行时数据]()结构。

3）在Java堆中生成一个代表这个类的java.lang.Class对象，作为方法区这些数据的访问入口。

虚拟机规范的这三点要求实际上并不具体，因此虚拟机实现与具体应用的灵活度相当大。例如“通过一个类的全限定名来获取定义此类的二进制字节流”，并没有指明二进制字节流要从一个Class文件中获取，准确地说是根本没有指明要从哪里获取及怎样获取。例如：

- 从ZIP包中读取，这很常见，最终成为日后JAR、EAR、WAR格式的基础。
- 从网络中获取，这种场景最典型的应用就是Applet。
- 运行时计算生成，这种场景使用得最多的就是动态代理技术，在java.lang.reflect.Proxy中，就是用了ProxyGenerator.generateProxyClass来为特定接口生成$Proxy的代理类的二进制字节流。
- 由其他文件生成，典型场景：JSP应用。
- 从数据库中读取，这种场景相对少见些，有些中间件服务器（如SAP Netweaver）可以选择把程序安装到数据库中来完成程序代码在集群间的分发。

相对于**`类加载`**过程的其他阶段，**`加载`**阶段（准确地说，是**`加载`**阶段中获取类的二进制字节流的动作）是开发期可控性最强的阶段，因为加载阶段既可以使用系统提供的**`Class Loader`**来完成，也可以由用户自定义的类加载器去完成，开发人员们可以通过定义自己的类加载器去控制字节流的获取方式。

**`加载`**阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，方法区中的数据存储格式由虚拟机实现自行定义，虚拟机规范未规定此区域的具体数据结构。然后在Java堆中实例化一个java.lang.Class类的对象，这个对象将作为程序访问方法区中的这些类型数据的外部接口。加载阶段与连接阶段的部分内容（如一部分字节码文件格式验证动作）是交叉进行的，加载阶段尚未完成，连接阶段可能已经开始，但这些夹在加载阶段之中进行的动作，仍然属于连接阶段的内容，这两个阶段的开始时间仍然保持着固定的先后顺序。

## 验证

验证是连接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

### **1.文件格式验证**

第一阶段要验证字节流是否符合Class文件格式的规范，并且能被当前版本的虚拟机处理。这一阶段可能包括下面这些验证点：

- 是否以魔数0xCAFEBABE开头。
- 主、次版本号是否在当前虚拟机处理范围之内。
- 常量池的常量中是否有不被支持的常量类型（检查常量tag标志）。
- 指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量。
- CONSTANT_Utf8_info型的常量中是否有不符合UTF8编码的数据。
- Class文件中各个部分及文件本身是否有被删除的或附加的其他信息。

实际上第一阶段的验证点还远不止这些，上面这些只是从HotSpot虚拟机源码 [[1\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00046.html#ch1-back) 中摘抄的一小部分，该验证阶段的主要目的是保证输入的字节流能正确地解析并存储于方法区之内，格式上符合描述一个Java类型信息的要求。

这阶段的验证是基于`字节流`进行的。

经过了这个阶段的验证之后，字节流才会进入内存的**`方法区`**中进行存储，所以后面的三个验证阶段全部是基于**`方法区`**的存储结构进行的。

### **2.元数据验证**

第二阶段是对字节码描述的信息进行语义分析，以保证其描述的信息符合Java语言规范的要求，这个阶段可能包括的验证点如下：

- ·这个类是否有父类（除了java.lang.Object之外，所有的类都应当有父类）。
- ·这个类的父类是否继承了不允许被继承的类（被final修饰的类）。
- ·如果这个类不是抽象类，是否实现了其父类或接口之中要求实现的所有方法。
- ·类中的字段、方法是否与父类产生了矛盾（例如覆盖了父类的final字段，或者出现不符合规则的方法重载，例如方法参数都一致，但返回值类型却不同等）。

第二阶段的主要目的是对类的元数据信息进行语义校验，保证不存在不符合Java语言规范的元数据信息。

### **3.字节码验证**

第三阶段是整个验证过程中最复杂的一个阶段，主要工作是进行数据流和控制流分析。在第二阶段对元数据信息中的数据类型做完校验后，这阶段将对类的方法体进行校验分析。这阶段的任务是保证被校验类的方法在运行时不会做出危害虚拟机安全的行为，例如：

- ·保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出现类似这样的情况：在操作栈中放置了一个int类型的数据，使用时却按long类型来加载入本地变量表中。
- ·保证跳转指令不会跳转到方法体以外的字节码指令上。
- ·保证方法体中的类型转换是有效的，例如可以把一个子类对象赋值给父类数据类型，这是安全的，但是把父类对象赋值给子类数据类型，甚至把对象赋值给与它毫无继承关系、完全不相干的一个数据类型，则是危险和不合法的。

如果一个类方法体的字节码没有通过字节码验证，那肯定是有问题的；但如果一个方法体通过了字节码验证，也不能说明其一定就是安全的。即使字节码验证之中进行了大量的检查，也不能保证这一点。这里涉及了离散数学中一个很著名的问题“Halting Problem” [[2\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00046.html#ch2-back) ：通俗一点的说法就是，通过程序去校验程序逻辑是无法做到绝对准确的——不能通过程序准确地检查出程序是否能在有限的时间之内结束运行。

### **4.符号引用验证**

最后一个阶段的校验发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作将在连接的第三个阶段——解析阶段中发生。符号引用验证可以看做是对类自身以外（常量池中的各种符号引用）的信息进行匹配性的校验，通常需要校验以下内容：

- ·符号引用中通过字符串描述的全限定名是否能找到对应的类。
- ·在指定类中是否存在符合方法的字段描述符及简单名称所描述的方法和字段。
- ·符号引用中的类、字段和方法的访问性（private、protected、public、default）是否可被当前类访问。

符号引用验证的目的是确保解析动作能正常执行，如果无法通过符号引用验证，将会抛出一个java.lang.IncompatibleClassChangeError异常的子类，如java.lang.IllegalAccessError、java.lang.NoSuchFieldError、java.lang.NoSuchMethodError等。

验证阶段对于虚拟机的类加载机制来说，是一个非常重要的、但不一定是必要的阶段。

## 准备(根据类变量的初始值(0值), 初始化)

**`准备`**阶段是正式为[类变量]()**分配内存**并设置**类变量初始值**的阶段，[这些内存都将在**`方法区`**中进行分配]()。

这个阶段中有两个容易产生混淆的概念需要强调一下，

- 这时候进行内存分配的仅包括`类变量`（被static修饰的变量），而不包括`实例变量`，`实例变量`将会在对象实例化时(猜测作者的意思是【初始化阶段】)随着对象一起分配在Java堆中。
- 这里所说的初始值“通常情况”下是**数据类型的零值**，假设一个类变量的定义为：

```
public static int value = 123;
```

> 那么[变量value在【准备阶段】过后的初始值为0而不是123，因为这时候尚未开始执行任何Java方法]()，而把value赋值为123的[putstatic]()指令是程序被编译后，存放于[类构造器<clinit>()方法]()之中，所以把value赋值为123的动作将在[【初始化阶段】]()才会被执行。

注意这里所说的初始值概念，比如一个类变量定义为：

```java
// 变量 v 在【准备阶段】过后的初始值为 0 而不是 8080
// 将 v 赋值为 8080 的 putstatic 指令是程序被编译后，存在类构造器<clinit>方法中
public static int v = 8080;
```

但如果声明为final所修饰时：

```java
// 在【编译阶段】会为变量 v 生成 ConstantValue 属性
// 在【准备阶段】，虚拟机会根据 ConstantValue 属性将 v 赋值为 8080
public static final int v = 8080;
```



## 解析

`解析`阶段是虚拟机将`常量池`内的符号引用替换为直接引用的过程，符号引用在前一章讲解Class文件格式的时候就已经出现过多次，在Class文件中它以CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info等类型的常量出现，那解析阶段中所说的直接引用与符号引用又有什么关联呢？

- 符号引用（Symbolic References）：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标并不一定已经加载到内存中。
- 直接引用（Direct References）：直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用是与虚拟机实现的内存布局相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经在内存中存在。

虚拟机规范之中并未规定解析阶段发生的具体时间，只要求了在执行anewarray、checkcast、getfield、getstatic、instanceof、invokeinterface、invokespecial、invokestatic、invokevirtual、multianewarray、new、putfield和putstatic这13个用于操作符号引用的字节码指令之前，先对它们所使用的符号引用进行解析。所以虚拟机实现会根据需要来判断，到底是在类被加载器加载时就对常量池中的符号引用进行解析，还是等到一个符号引用将要被使用前才去解析它。

对同一个符号引用进行多次解析请求是很常见的事情，虚拟机实现可能会对第一次解析的结果进行缓存（在运行时常量池中记录直接引用，并把常量标识为已解析状态）从而避免解析动作重复进行。无论是否真正执行了多次解析动作，虚拟机需要保证的都是在同一个实体中，如果一个符号引用之前已经被成功解析过，那么后续的引用解析请求就应当一直成功；同样地，如果第一次解析失败了，其他指令对这个符号的解析请求也应该收到相同的异常。

解析动作主要针对类或接口、字段、类方法、接口方法四类符号引用进行，分别对应于常量池的CONSTANT_Class_info、CONSTANT_Fieldref_info、CONSTANT_Methodref_info及CONSTANT_InterfaceMethodref_info四种常量类型 [[3\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00046.html#ch3-back) 。下面将讲解这四种引用的解析过程。

### 1.类或接口的解析

假设当前代码所处的类为D，如果要把一个从未解析过的符号引用N解析为一个类或接口C的直接引用，那虚拟机完成整个解析的过程需要包括以下3个步骤：

1）如果C不是一个数组类型，那虚拟机将会把代表N的全限定名传递给D的类加载器去加载这个类C。在加载过程中，由于无数据验证、字节码验证的需要，又将可能触发其他相关类的加载动作，例如加载这个类的父类或实现的接口。一旦这个加载过程出现了任何异常，解析过程就将宣告失败。

2）如果C是一个数组类型，并且数组的元素类型为对象，也就是N的描述符会是类似“[Ljava.lang.Integer”的形式，那将会按照第1点的规则加载数组元素类型。如果N的描述符如前面所假设的形式，需要加载的元素类型就是“java.lang.Integer”，接着由虚拟机生成一个代表此数组维度和元素的数组对象。

3）如果上面的步骤没有出现任何异常，那么C在虚拟机中实际上已经成为一个有效的类或接口了，但在解析完成之前还要进行符号引用验证，确认C是否具备对D的访问权限。如果发现不具备访问权限，将抛出java.lang.IllegalAccessError异常。

### 2.字段解析

要解析一个未被解析过的字段符号引用，首先将会对字段表内class_index [[4\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00046.html#ch4-back) 项中索引的CONSTANT_Class_info符号引用进行解析，也就是字段所属的类或接口的符号引用。如果在解析这个类或接口符号引用的过程中出现了任何异常，都会导致字段符号引用解析的失败。如果解析成功完成，那将这个字段所属的类或接口用C表示，虚拟机规范要求按照如下步骤对C进行后续字段的搜索：

1）如果C本身就包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。

2）否则，如果在C中实现了接口，将会按照继承关系从上往下递归搜索各个接口和它的父接口，如果接口中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。

3）否则，如果C不是java.lang.Object的话，将会按照继承关系从上往下递归搜索其父类，如果在父类中包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。

4）否则，查找失败，抛出java.lang.NoSuchFieldError异常。

如果查找过程成功返回了引用，将会对这个字段进行权限验证，如果发现不具备对字段的访问权限，将抛出java.lang.IllegalAccessError异常。

在实际应用中，虚拟机的编译器实现可能会比上述规范要求得更加严格一些，如果有一个同名字段同时出现在C的接口和父类中，或者同时在自己或父类的多个接口中出现，那编译器将可能拒绝编译。在代码清单7-4中，如果注释了Sub类中的“public static int A=4；”，接口与父类同时存在字段A，那编译器将提示“The field Sub.A is ambiguous”，并且会拒绝编译这段代码。

代码清单7-4　字段解析

------

```
package org.fenixsoft.classloading;
public class FieldResolution {
　interface Interface0 {
　　int A = 0;
　}
　interface Interface1 extends Interface0 {
　　int A = 1;
　}
　interface Interface2 {
　　int A = 2;
　}
　static class Parent implements Interface1 {
　　public static int A = 3;
　}
　static class Sub extends Parent implements Interface2 {
　　public static int A = 4;
　}
　public static void main(String[] args) {
　　System.out.println(Sub.A);
　}
}
```

------

### 3.类方法解析

类方法解析的第一个步骤与字段解析一样，也是需要先解析出类方法表的class_index [[5\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00046.html#ch5-back) 项中索引的方法所属的类或接口的符号引用，如果解析成功，我们依然用C表示这个类，接下来虚拟机将会按照如下步骤进行后续的类方法搜索：

1）类方法和接口方法符号引用的常量类型定义是分开的，如果在类方法表中发现class_index中索引的C是个接口，那就直接抛出java.lang.IncompatibleClassChangeError异常。

2）如果通过了第（1）步，在类C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。

3）否则，在类C的父类中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。

4）否则，在类C实现的接口列表及它们的父接口之中递归查找是否有简单名称和描述符都与目标相匹配的方法，如果存在匹配的方法，说明类C是一个抽象类，这时候查找结束，抛出java.lang.AbstractMethodError异常。

5）否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError。

最后，如果查找过程成功返回了直接引用，将会对这个方法进行权限验证；如果发现不具备对此方法的访问权限，将抛出java.lang.IllegalAccessError异常。

### 4.接口方法解析

接口方法也是需要先解析出接口方法表的class_index [[6\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00046.html#ch6-back) 项中索引的方法所属的类或接口的符号引用，如果解析成功，依然用C表示这个接口，接下来虚拟机将会按照如下步骤进行后续的接口方法搜索：

1）与类方法解析相反，如果在接口方法表中发现class_index中的索引C是个类而不是接口，那就直接抛出java.lang.IncompatibleClassChangeError异常。

2）否则，在接口C中查找是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。

3）否则，在接口C的父接口中递归查找，直到java.lang.Object类（查找范围会包括Object类）为止，看是否有简单名称和描述符都与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。

4）否则，宣告方法查找失败，抛出java.lang.NoSuchMethodError异常。

由于接口中的所有方法都默认是public的，所以不存在访问权限的问题，因此接口方法的符号解析应当不会抛出java.lang.IllegalAccessError异常。

## 初始化(根据程序员，进行初始化)

`初始化`是`类加载`过程的最后一步，前面的`类加载`过程中，除了在`加载`阶段用户应用程序可以通过`自定义类加载器`参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行**类中定义的Java程序代码**（或者说是字节码）。

在`准备`阶段，变量已经赋过一次**系统要求的初始值**。

在`初始化`阶段，则是**根据程序员通过程序制定的主观计划去初始化类变量**和其他资源，或者可以从另外一个角度来表达：初始化阶段是执行`类构造器`<clinit>()方法的过程。

### JVM初始化类的步骤

1. 若该类还没有被加载和连接，则程序先加载并连接该类
2. 若该类的父类还没有初始化，则先初始化该类的父类
3. 若该类种有静态代码块，则系统依次执行这些代码块

### 执行`类构造器`<clinit>()方法

我们先看一下<clinit>()方法执行过程中可能会影响程序运行行为的一些特点和细节[[7\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00046.html#ch7-back) ：

- <clinit>()方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，**编译器收集的顺序是由语句在源文件中出现的顺序所决定的**，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块中可以赋值，但是不能访问。

- <clinit>()方法与<u>类的构造函数</u>，或者说<u>实例构造器</u><init>()方法，不同<!--非常重要，也是非常容易混淆的点-->
  它不需要显式地调用父类构造器，虚拟机会保证在子类的<clinit>()方法执行之前，父类的<clinit>()方法已经执行完毕。因此在虚拟机中第一个被执行的<clinit>()方法的类肯定是java.lang.Object。

- 由于父类的<clinit>()方法先执行，也就意味着**父类中定义的`静态语句块`要优先于子类的变量赋值操作**，如代码清单7-5中，字段B的值将会是2而不是1。

代码清单7-5　<clinit>()方法执行顺序

------

```java
static class Parent {
    public static int A = 1;
    static {
    A = 2;
    }
}
static class Sub extends Parent {
    public static int B = A;
}
public static void main(String[] args) {
    System.out.println(Sub.B);
}
```

------

- <clinit>()方法对于类或接口来说并不是必须的，如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成<clinit>()方法。

- 接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样都会生成<clinit>()方法。但接口与类不同的是，执行接口的<clinit>()方法不需要先执行父接口的<clinit>()方法。只有当父接口中定义的变量被使用时，父接口才会被初始化。另外，接口的实现类在初始化时也一样不会执行接口的<clinit>()方法。

- 虚拟机会保证一个类的<clinit>()方法在多线程环境中被正确地加锁和同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的<clinit>()方法，其他线程都需要阻塞等待，直到活动线程执行<clinit>()方法完毕。如果在一个类的<clinit>()方法中有耗时很长的操作，那就可能造成多个进程阻塞，在实际应用中这种阻塞往往是很隐蔽的。代码清单7-6演示了这种场景。

代码清单7-6　字段解析

------

```java
static class DeadLoopClass {
    static {
        // 如果不加上这个if语句，编译器将提示"Initializer does not complete normally"并拒绝编译
        if (true) {
            System.out.println(Thread.currentThread() + "init DeadLoopClass");
            while (true) {
            }
        }
    }
}
public static void main(String[] args) {
    Runnable script = new Runnable() {
        public void run() {
            System.out.println(Thread.currentThread() + "start");
            DeadLoopClass dlc = new DeadLoopClass();
            System.out.println(Thread.currentThread() + " run over");
        }
    };
    Thread thread1 = new Thread(script);
    Thread thread2 = new Thread(script);
    thread1.start();
    thread2.start();
}
```

------

运行结果如下，一条线程正在死循环以模拟长时间操作，另外一条线程在阻塞等待：

------

```java
Thread[Thread-0,5,main]start
Thread[Thread-1,5,main]start
Thread[Thread-0,5,main]init DeadLoopClass
```

------

# 类加载器

虚拟机设计团队把类加载阶段中的“通过一个类的全限定名来获取描述此类的二进制字节流”这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。实现这个动作的代码模块被称为“类加载器”。

## 类与类加载器

比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提之下才有意义，否则，即使这两个类是来源于同一个Class文件，只要加载它们的类加载器不同，那这两个类就必定不相等。

这里所指的“相等”，包括代表类的Class对象的equals()方法、isAssignableFrom()方法、isInstance()方法的返回结果，也包括了使用instanceof关键字做对象所属关系判定等情况。

如果没有注意到类加载器的影响，在某些情况下可能会产生具有迷惑性的结果。

代码清单7-7　不同的类加载器对instanceof关键字运算结果的影响

------

```java
/**
 * 类加载器与instanceof关键字演示
 * 
 * @author zzm
 */
public class ClassLoaderTest {
    public static void main(String[] args) throws Exception {
        ClassLoader myLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1) + 
　　".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
            }
        };
        Object obj = myLoader.loadClass("org.fenixsoft.classloading.
　ClassLoaderTest").newInstance();
        System.out.println(obj.getClass());
        System.out.println(obj instanceof org.fenixsoft.classloading.
　ClassLoaderTest);
    }
}
```

------

运行结果：

------

```
class org.fenixsoft.classloading.ClassLoaderTest
false
```

------

代码清单7-7中构造了一个简单的类加载器。

它可以加载与自己在同一路径下的Class文件。

两行输出结果中，从第一句可以看到这个对象确实是类org.fenixsoft.classloading.ClassLoaderTest实例化出来的对象，但从第二句可以发现这个对象与类org.fenixsoft.classloading.ClassLoaderTest做所属类型检查的时候却返回了false，这是因为虚拟机中存在了两个ClassLoaderTest类，一个是由系统应用程序类加载器加载的，另外一个是由我们自定义的类加载器加载的，虽然都来自同一个Class文件，但依然是两个独立的类，做对象所属类型检查时结果自然为false。

## 双亲委派模型

站在Java虚拟机的角度讲，只存在两种不同的类加载器：

- 一种是启动类加载器（Bootstrap ClassLoader），这个类加载器使用C++语言实现  ，是虚拟机自身的一部分；
- 另外一种就是所有其他的类加载器，这些类加载器都由Java语言实现，独立于虚拟机外部，并且全都继承自抽象类java.lang.ClassLoader。

从Java开发人员的角度来看，类加载器就还可以划分得更细致一些：

- 启动类加载器（Bootstrap ClassLoader）：前面已经介绍过，这个类加载器负责将存放在<JAVA_HOME>\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如rt.jar，名字不符合的类库即使放在lib目录中也不会被加载）类库加载到虚拟机内存中。启动类加载器无法被Java程序直接引用。

- 扩展类加载器（Extension ClassLoader）：这个加载器由sun.misc.Launcher$ExtClassLoader实现，它负责加载<JAVA_HOME>\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器。

- 应用程序类加载器（Application ClassLoader）：这个类加载器由sun.misc.Launcher$AppClassLoader来实现。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以一般也称它为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。


我们的应用程序都是由这三种类加载器互相配合进行加载的，如果有必要，还可以加入自己定义的类加载器。这些类加载器之间的关系一般会如图7-2所示。

<img src="https://zoctan.github.io/2018/07/25/Java/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E7%9A%84%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE%E6%A8%A1%E5%9E%8B.jpg" alt="类加载器的双亲委派模型" style="zoom:75%;" />

图7-2　类加载器双亲委派模型

图7-2中所展示的类加载器之间的这种层次关系，就称为类加载器的双亲委派模型（Parents Delegation Model）。双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。

这里类加载器之间的父子关系一般不会以继承（Inheritance）的关系来实现，而是都使用组合（Composition）关系来复用父加载器的代码。

类加载器的双亲委派模型在JDK 1.2期间被引入并被广泛应用于之后几乎所有的Java程序中，但它并不是一个强制性的约束模型，而是Java设计者们推荐给开发者们的一种类加载器实现方式。

双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。<!--说白了就是不属于java的基础类，java不加载-->

使用双亲委派模型来组织类加载器之间的关系，有一个显而易见的好处就是Java类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存放在rt.jar之中，

- 无论哪一个类加载器要加载这个类，最终都是委派给启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。
- 相反，如果没有使用双亲委派模型，由各个类加载器自行去加载的话，如果用户自己写了一个名为java.lang.Object的类，并放在程序的ClassPath中，那系统中将会出现多个不同的Object类，Java类型体系中最基础的行为也就无从保证，应用程序也将会变得一片混乱。<!--如果您有兴趣的话，可以尝试去写一个与rt.jar类库中已有类重名的Java类，将会发现可以正常编译，但永远无法被加载运行-->

双亲委派模型对于保证Java程序的稳定运作很重要，但它的实现却非常简单，实现双亲委派的代码都集中在java.lang.ClassLoader的loadClass()方法之中，如代码清单7-8所示，逻辑清晰易懂：先检查是否已经被加载过，若没有加载则调用父加载器的loadClass()方法，若父加载器为空则默认使用启动类加载器作为父加载器。如果父类加载失败，则在抛出ClassNotFoundException异常后，再调用自己的findClass()方法进行加载。

代码清单7-8　双亲委派模型的实现

------

```
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
{
    // 首先，检查请求的类是否已经被加载过了
    Class c = findLoadedClass(name);
    if (c == null) {
        try {
        if (parent != null) {
            c = parent.loadClass(name, false);
        } else {
            c = findBootstrapClassOrNull(name);
        }
        } catch (ClassNotFoundException e) {
            // 如果父类加载器抛出ClassNotFoundException
            // 则说明父类加载器无法完成加载请求
        }
        if (c == null) {
            // 在父类加载器无法加载的时候
            // 再调用本身的findClass方法来进行类加载
            c = findClass(name);
        }
    }
    if (resolve) {
        resolveClass(c);
    }
    return c;
}
```

------

## 破坏双亲委派模型

到现在为止，双亲委派模型主要出现过三次较大规模的“被破坏”情况。

双亲委派模型的第一次“被破坏”其实发生在双亲委派模型出现之前——即JDK 1.2发布之前。由于双亲委派模型在JDK 1.2之后才被引入的，而类加载器和抽象类java.lang.ClassLoader则在JDK 1.0时代就已经存在，面对已经存在的用户自定义类加载器的实现代码，Java设计者们引入双亲委派模型时不得不做出一些妥协。为了向前兼容，JDK 1.2之后的java.lang.ClassLoader添加了一个新的protected方法findClass()，在此之前，用户去继承java.lang.ClassLoader的唯一目的就是为了重写loadClass()方法，因为虚拟机在进行类加载的时候会调用加载器的私有方法loadClassInternal()，而这个方法的唯一逻辑就是去调用自己的loadClass()。

上一节我们已经看过loadClass()方法的代码，双亲委派的具体逻辑就实现在这个方法之中，JDK 1.2之后已不提倡用户再去覆盖loadClass()方法，而应当把自己的类加载逻辑写到findClass()方法中，在loadClass()方法的逻辑里如果父类加载失败，则会调用自己的findClass()方法来完成加载，这样就可以保证新写出来的类加载器是符合双亲委派规则的。

双亲委派模型的第二次“被破坏”是由这个模型自身的缺陷所导致的，双亲委派很好地解决了各个类加载器的基础类的统一问题（越基础的类由越上层的加载器进行加载），基础类之所以被称为“基础”，是因为它们总是作为被用户代码调用的API，但世事往往没有绝对的完美，如果基础类又要调用回用户的代码，那该怎么办了？

这并非是不可能的事情，一个典型的例子便是JNDI服务，JNDI现在已经是Java的标准服务，它的代码由启动类加载器去加载（在JDK 1.3时代放进去的rt.jar），但JNDI的目的就是对资源进行集中管理和查找，它需要调用由独立厂商实现并部署在应用程序的ClassPath下的JNDI接口提供者（SPI，Service Provider Interface）的代码，但启动类加载器不可能“认识”这些代码啊！那该怎么办？

<!--解决方法-->

为了解决这个困境，Java设计团队只好引入了一个不太优雅的设计：线程上下文类加载器（Thread Context ClassLoader）。这个类加载器可以通过java.lang.Thread类的setContextClassLoaser()方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个；如果在应用程序的全局范围内都没有设置过，那么这个类加载器默认就是应用程序类加载器。

有了线程上下文类加载器，就可以做一些“舞弊”的事情了，JNDI服务使用这个线程上下文类加载器去加载所需要的SPI代码，也就是父类加载器请求子类加载器去完成类加载的动作，这种行为实际上就是打通了双亲委派模型的层次结构来逆向使用类加载器，已经违背了双亲委派模型的一般性原则，但这也是无可奈何的事情。Java中所有涉及SPI的加载动作基本上都采用这种方式，例如JNDI、JDBC、JCE、JAXB和JBI等。

双亲委派模型的第三次“被破坏”是由于用户对程序动态性的追求而导致的，这里所说的“动态性”指的是当前一些非常“热”门的名词：代码热替换（HotSwap）、模块热部署（Hot Deployment）等，对于一些生产系统来说，关机重启一次可能就要被列为生产事故，这种情况下热部署就对软件开发者，尤其是企业级软件开发者具有很大的吸引力。

在JSR-297 [[4\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00047.html#ch4-back) 、JSR-277 [[5\]](http://reader.epubee.com/books/mobile/11/110629db113c9cb59f62032f449dd46c/text00047.html#ch5-back) 规范从纸上标准变成真正可运行的程序之前，OSGi是当前业界“事实上”的Java模块化标准，而OSGi实现模块化热部署的关键则是它自定义的类加载器机制的实现。每一个程序模块（OSGi中称为Bundle）都有一个自己的类加载器，当需要更换一个Bundle时，就把Bundle连同类加载器一起换掉以实现代码的热替换。

在OSGi环境下，类加载器不再是双亲委派模型中的树状结构，而是进一步发展为网状结构，当收到类加载请求时，OSGi将按照下面的顺序进行类搜索：

（1）将以java.*开头的类，委派给父类加载器加载。

（2）否则，将委派列表名单内的类，委派给父类加载器加载。

（3）否则，将Import列表中的类，委派给Export这个类的Bundle的类加载器加载。

（4）否则，查找当前Bundle的ClassPath，使用自己的类加载器加载。

（5）否则，查找类是否在自己的Fragment Bundle中，如果在，则委派给Fragment Bundle的类加载器加载。

（6）否则，查找Dynamic Import列表的Bundle，委派给对应Bundle的类加载器加载。

（7）否则，类查找失败。

上面的查找顺序中只有开头两点仍然符合双亲委派规则，其余的类查找都是在平级的类加载器中进行的。

笔者虽然使用了“被破坏”这个词来形容上述不符合双亲委派模型原则的行为，但这里“被破坏”并不带有贬义的感情色彩。只要有足够意义和理由，突破已有的原则就可算作一种创新。正如OSGi中的类加载器并不符合传统的双亲委派的类加载器，并且业界对其为了实现热部署而带来的额外的高复杂度还存在不少争议，但在Java程序员中基本有一个共识：OSGi中对类加载器的使用是很值得学习的，弄懂了OSGi的实现，自然就明白了类加载器的精粹。







