---
title: JDK-IO-Serializable
date: 2021-05-28 16:43:54
tags:
---

## Java序列化与反序列化

- 序列化：将对象写入到IO流中
- 反序列化：从IO流中恢复对象

序列化机制允许将实现序列化的Java对象转换为字节序列，这些字节序列可以保存在磁盘上，或通过网络传输，以达到以后恢复成原来的对象。序列化机制使得对象可以脱离程序的运行而独立存在。

要想有序列化的能力，得实现`Serializable`接口

```
public class SerializableTest implements Serializable {
    private static final long serialVersionUID = -3751255153289772365L;
}
```

JVM会在运行时判断类的`serialVersionUID`来验证版本一致性，如果传来的字节流中的serialVersionUID与本地相应类的serialVersionUID相同则认为是一致的，可以进行反序列化，否则就会出现序列化版本不一致的异常。

```java
java.io.InvalidClassException:com.taobao.query.TestSerializable;
 local class incompatible: stream classdesc serialVersionUID = -7165097063094245447,local class    serialVersionUID = 6678378625230229450
```

## Jackson序列化



## Dubbo与序列化

<img src="https://mmbiz.qpic.cn/mmbiz_jpg/R7PtjL3tdAickVUF9IZmibFdmF1FlgL2NuBu5TXia3HmibyicxYfMkYv7YzoGKy4ISEs8MEs3CjMw28JdxibnibFHCPfA/640" alt="Image" style="zoom:67%;" />

图片来源：https://dubbo.apache.org/zh/docs/v2.7/dev/design/

从Dubbo的调用链可以发现是有一个序列化节点的，其支持的序列化协议一共有四种：

1. dubbo序列化：阿里尚未开发成熟的高效java序列化实现，阿里不建议在生产环境使用它
2. hessian2序列化：hessian是一种跨语言的高效二进制序列化方式。但这里实际不是原生的hessian2序列化，而是阿里修改过的hessian lite，它是dubbo RPC默认启用的序列化方式
3. json序列化：目前有两种实现，一种是采用的阿里的fastjson库，另一种是采用dubbo中自己实现的简单json库，但其实现都不是特别成熟，而且json这种文本序列化性能一般不如上面两种二进制序列化。
4. java序列化：主要是采用JDK自带的Java序列化实现，性能很不理想。

如果在RPC中使用了Java序列化，那下面的这三个坑一定注意不要踩





### 类实现`Serializable`接口但没有指定`serialVersionUID`

如果实现了`Serializable`的类没有指定`serialVersionUID`，编译器编译的时候会根据类名、接口名、成员方法及属性等来生成一个64位的哈希字段，这就决定了这个类在序列化上一定不是向前兼容的

假如我们先有`Student`这样的一个类

```
public class Student implements Serializable {

    private static int startId = 1000;

    private int id;

    public Student() {
        id = startId ++;
    }
}
```

我们将其序列化到磁盘：

```java
private static void serialize() {
    try {

        Student student = new Student();
        FileOutputStream fileOut =
                new FileOutputStream("/tmp/student.ser");
        ObjectOutputStream out = new ObjectOutputStream(fileOut);
        out.writeObject(student);
        out.close();
        fileOut.close();
        System.out.printf("Serialized data is saved in /tmp/student.ser");
    } catch (
            IOException i) {
        i.printStackTrace();
    }
}
```

然后给`Student`类加一个字段

```java
public class Student implements Serializable {

    private static int startId = 1000;

    private int id;
  // 注意这里我们已经加了一个属性
    private String name;

    public Student() {
        id = startId ++;
    }
}
```

我们再去解码，发现程序会抛出异常：

```
java.io.InvalidClassException: com.idealism.base.Student; local class incompatible: stream classdesc serialVersionUID = -1534228028811562580, local class serialVersionUID = 630353564791955009
 at java.io.ObjectStreamClass.initNonProxy(ObjectStreamClass.java:699)
 at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:2001)
 at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1848)
 at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2158)
 at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1665)
 at java.io.ObjectInputStream.readObject(ObjectInputStream.java:501)
 at java.io.ObjectInputStream.readObject(ObjectInputStream.java:459)
 at com.idealism.base.SerializableTest.deserialize(SerializableTest.java:34)
 at com.idealism.base.SerializableTest.main(SerializableTest.java:9)
```

根因是RPC的参数实现了`Serializable`接口，但是没有指定`serialVersionUID`，编译器会根据类名、接口名、成员方法及属性等来生成一个64位的哈希字段，当服务端类升级之后导致了服务端发送给客户端的字节流中的`serialVersionUID`发生了改变，因此当客户端反序列化去检查`serialVersionUID`字段的时候发现发生了变化被判定了异常。

### 父类实现`Serializable`接口，并且指定了`serialVersionUID`但是子类没有指定`serialVersionUID`

我们对前面的例子中的`Student`类稍微改一下

```
public class Student extends Base{

    private static int startId = 1000;

    private int id;

    public Student() {
        id = startId ++;
    }
}
```

其中父类长这样：

```
public class Base implements Serializable {
    private static final long serialVersionUID = 218886242758597651L;

    private Date gmtCreate;
}
```

如果我们按照之前的讨论在本地进行一次序列化和反序列化，程序依然抛异常：

```
java.io.InvalidClassException: com.idealism.base.Student; local class incompatible: stream classdesc serialVersionUID = 1049562984784675762, local class serialVersionUID = 7566357243685852874
 at java.io.ObjectStreamClass.initNonProxy(ObjectStreamClass.java:699)
 at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:2001)
 at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1848)
 at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2158)
 at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1665)
 at java.io.ObjectInputStream.readObject(ObjectInputStream.java:501)
 at java.io.ObjectInputStream.readObject(ObjectInputStream.java:459)
 at com.idealism.base.SerializableTest.deserialize(SerializableTest.java:34)
 at com.idealism.base.SerializableTest.main(SerializableTest.java:9)
```

我们在设计类的时候公共属性要放到基类，这条经验指导放到这个case中仍然不太正确，而且这个case比上一个还要隐蔽，问题是子类中的`serialVersionUID`仍然是编译器自动生成的。当然可以把父类中`serialVersionUID`的改为非`private`来解这个问题，不过我仍然建议每个有序列化需求的类都显式指定`serialVersionUID`的值。

如果序列化遇到类之间的组合或者继承关系，则Java按照下面的规则处理：

- 当一个对象的实例变量引用其他对象，序列化该对象时也把引用对象进行序列化，而不管其是否实现了`Serializable`接口
- 如果子类实现了`Serializable`，则序列化时只序列化子类，不会序列化父类中的属性
- 如果父类实现了`Serializable`，则序列化时子类和父类都会被序列化，异常场景如本例所指

还有一点要注意：如果类的实例中有静态变量，改属性不会被序列化和反序列化

### 类中有枚举值

《阿里巴巴开发规约》中有这么一条：

> 【强制】二方库例可以定义枚举类型，参数可以使用枚举类型，但是接口返回值不允许使用枚举类型或者包含枚举类型的POJO对象。
>
> 说明：由于升级原因，导致双方的枚举类不尽相同，在接口解析，类反序列化时出现异常

这里会出现这样一个限制的原因是Java对枚举的序列化和反序列化采用完全不同的策略。序列化的结果中仅包含枚举的名字，而不包含枚举的具体定义，反序列化的时候客户端从序列化结果中读取枚举的name，然后调用`java.lang.Enum#valueOf`根据本地的枚举定义获取具体的枚举值。

我们仍然用之前的代码举例：

```
public class Student implements Serializable {

    private static final long serialVersionUID = 2528736437985230667L;
    
    private static int startId = 1000;

    private int id;

    private String name;
    // 新增字段，校服尺码，其类型是一个枚举
    private SchoolUniformSizeEnum schoolUniformSize;

    public Student() {
        id = startId ++;
    }
}
```

假如学生这个类中新增了一个校服尺码的枚举值

```
public enum SchoolUniformSizeEnum {
    SMALL,
    MEDIUM,
    LARGE
}
```

假如服务端此时对这个枚举进行了升级，但是客户端的二方包中仍然只有三个值：

```
public enum SchoolUniformSizeEnum {
    SMALL,
    MEDIUM,
    LARGE,
    OVERSIZED
}
```

如果服务端有逻辑给客户端返回了这个新增的枚举值：

```
private static void serialize() {
    try {
        Student student = new Student();
        // 服务端升级了枚举
        student.setSchoolUniformSize(SchoolUniformSizeEnum.OVERSIZED);
        FileOutputStream fileOut =
                new FileOutputStream("/tmp/student.ser");
        ObjectOutputStream out = new ObjectOutputStream(fileOut);
        out.writeObject(student);
        out.close();
        fileOut.close();
        System.out.printf("Serialized data is saved in /tmp/student.ser");
    } catch (
            IOException i) {
        i.printStackTrace();
    }
}
```

因为客户端的包还没有升级，所以当客户端读到这个新的字节流并序列化的时候会因为找不到对应的枚举值而抛异常。

```
java.io.InvalidObjectException: enum constant OVERSIZED does not exist in class com.idealism.base.SchoolUniformSizeEnum
 at java.io.ObjectInputStream.readEnum(ObjectInputStream.java:2130)
 at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1659)
 at java.io.ObjectInputStream.defaultReadFields(ObjectInputStream.java:2403)
 at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:2327)
 at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2185)
 at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1665)
 at java.io.ObjectInputStream.readObject(ObjectInputStream.java:501)
 at java.io.ObjectInputStream.readObject(ObjectInputStream.java:459)
 at com.idealism.base.SerializableTest.deserialize(SerializableTest.java:36)
 at com.idealism.base.SerializableTest.main(SerializableTest.java:9)
```