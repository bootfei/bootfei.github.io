---
title: mybatis-方法不能重载的原因
date: 2021-06-07 18:28:59
tags:
---

## 为什么不能重载？

- Springboot与Mybatis会有一个启动器的自动配置类`MybatisAutoConfiguration`，其中有一段代码就是创建`sqlSessionFactory`，如下图：
  ![img](https://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2dpdGVlLmNvbS9jaGVuamlhYmluZzY2Ni9CbG9nLWZpbGUvcmF3L21hc3Rlci9NeWJhaXRzJUU0JUI4JUFEJUU3JTlBJTg0JUU2JTk2JUI5JUU2JUIzJTk1JUU0JUI4JUJBJUU0JUJCJTgwJUU0JUI5JTg4JUU0JUI4JThEJUU4JTgzJUJEJUU5JTg3JThEJUU4JUJEJUJEJUVGJUJDJTlGLzIucG5n.jpg)
- 既然是创建失败，那么肯定是这里出现异常了，这里的**「大致思路」**就是：

> ❝
>
> 解析`XML`文件和`Mapper`接口，将Mapper中的方法与XML文件中`<select>`、`<insert>`等标签一一对应，那么Mapper中的方法如何与XML中`<select>`这些标签对应了，当然是唯一的`id`对应了，具体如何这个`id`的值是什么，如何对应？下面一一讲解。
>
> ❞

- 如上图的`SqlSessionFactory`的创建过程中，前面的部分代码都是设置一些配置，并没有涉及到解析XML的内容，因此答案肯定是在最后一行`return factory.getObject();`，于是此处打上断点，一点点看。于是一直到了`org.mybatis.spring.SqlSessionFactoryBean#buildSqlSessionFactory`这个方法中，其中一段代码如下：
  ![img](https://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2dpdGVlLmNvbS9jaGVuamlhYmluZzY2Ni9CbG9nLWZpbGUvcmF3L21hc3Rlci9NeWJhaXRzJUU0JUI4JUFEJUU3JTlBJTg0JUU2JTk2JUI5JUU2JUIzJTk1JUU0JUI4JUJBJUU0JUJCJTgwJUU0JUI5JTg4JUU0JUI4JThEJUU4JTgzJUJEJUU5JTg3JThEJUU4JUJEJUJEJUVGJUJDJTlGLzMucG5n.jpg)
- 略过不重要的代码，在`org.apache.ibatis.builder.xml.XMLMapperBuilder#configurationElement`这个方法中有一行重要的代码，如下图：
  ![img](https://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2dpdGVlLmNvbS9jaGVuamlhYmluZzY2Ni9CbG9nLWZpbGUvcmF3L21hc3Rlci9NeWJhaXRzJUU0JUI4JUFEJUU3JTlBJTg0JUU2JTk2JUI5JUU2JUIzJTk1JUU0JUI4JUJBJUU0JUJCJTgwJUU0JUI5JTg4JUU0JUI4JThEJUU4JTgzJUJEJUU5JTg3JThEJUU4JUJEJUJEJUVGJUJDJTlGLzQucG5n.jpg)
- 到`org.apache.ibatis.builder.MapperBuilderAssistant#addMappedStatement`这个方法返回值就是`MappedStatement`，不用多说，肯定是这个方法了，仔细一看，很清楚的看到了构建`id`的代码，如下图：
  ![img](https://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2dpdGVlLmNvbS9jaGVuamlhYmluZzY2Ni9CbG9nLWZpbGUvcmF3L21hc3Rlci9NeWJhaXRzJUU0JUI4JUFEJUU3JTlBJTg0JUU2JTk2JUI5JUU2JUIzJTk1JUU0JUI4JUJBJUU0JUJCJTgwJUU0JUI5JTg4JUU0JUI4JThEJUU4JTgzJUJEJUU5JTg3JThEJUU4JUJEJUJEJUVGJUJDJTlGLzUucG5n.jpg)
- 从上图可以知道，创建`id`的代码就是`id = applyCurrentNamespace(id, false);`，具体实现如下图：
  ![img](https://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2dpdGVlLmNvbS9jaGVuamlhYmluZzY2Ni9CbG9nLWZpbGUvcmF3L21hc3Rlci9NeWJhaXRzJUU0JUI4JUFEJUU3JTlBJTg0JUU2JTk2JUI5JUU2JUIzJTk1JUU0JUI4JUJBJUU0JUJCJTgwJUU0JUI5JTg4JUU0JUI4JThEJUU4JTgzJUJEJUU5JTg3JThEJUU4JUJEJUJEJUVGJUJDJTlGLzYucG5n.jpg)

> ❝
>
> 上图的代码已经很清楚了，`MappedStatement`中的`id=Mapper的全类名+'.'+方法名`。如果重载话，肯定会存在`id`相同的`MappedStatement`。
>
> ❞

- 到了这其实并不能说明方法不能重载啊，重复就重复呗，并没有冲突啊。这里需要看一个结构，如下：

```
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
      .conflictMessageProducer((savedValue, targetValue) ->
          ". please check " + savedValue.getResource() + " and " + targetValue.getResource());
```

- 构建好的`MappedStatement`都会存入`mappedStatements`中，如下代码：

```
public void addMappedStatement(MappedStatement ms) {
    //key 是id
    mappedStatements.put(ms.getId(), ms);
  }
```

- `StrictMap`的`put(k,v)`方法如下图：
  ![img](https://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2dpdGVlLmNvbS9jaGVuamlhYmluZzY2Ni9CbG9nLWZpbGUvcmF3L21hc3Rlci9NeWJhaXRzJUU0JUI4JUFEJUU3JTlBJTg0JUU2JTk2JUI5JUU2JUIzJTk1JUU0JUI4JUJBJUU0JUJCJTgwJUU0JUI5JTg4JUU0JUI4JThEJUU4JTgzJUJEJUU5JTg3JThEJUU4JUJEJUJEJUVGJUJDJTlGLzcucG5n.jpg)

## 如何找到XML中对应的SQL？

- 在使用Mybatis的时候只是简单的调用Mapper中的方法就可以执行SQL，如下代码：

```
List<UserInfo> userInfos = userMapper.selectList(Arrays.asList("192","198"));
```

> 一行简单的调用到底如何找到对应的SQL呢？其实就是根据`id`从`Map<String, MappedStatement> mappedStatements`中查找对应的`MappedStatement`。

- 在`org.apache.ibatis.session.defaults.DefaultSqlSession#selectList`方法有这一行代码如下图：
  ![img](https://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2dpdGVlLmNvbS9jaGVuamlhYmluZzY2Ni9CbG9nLWZpbGUvcmF3L21hc3Rlci9NeWJhaXRzJUU0JUI4JUFEJUU3JTlBJTg0JUU2JTk2JUI5JUU2JUIzJTk1JUU0JUI4JUJBJUU0JUJCJTgwJUU0JUI5JTg4JUU0JUI4JThEJUU4JTgzJUJEJUU5JTg3JThEJUU4JUJEJUJEJUVGJUJDJTlGLzgucG5n.jpg)
- `MappedStatement ms = configuration.getMappedStatement(statement);`这行代码就是根据`id`从`mappedStatements`获取对应的`MappedStatement`，源码如下:

```
public MappedStatement getMappedStatement(String id) {
    return this.getMappedStatement(id, true);
  }
```