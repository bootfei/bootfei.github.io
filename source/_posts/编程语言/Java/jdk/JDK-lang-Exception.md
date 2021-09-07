---
title: JDK-lang-Exception
date: 2021-09-07 18:28:08
tags:
---





## 应用

### RuntimeException应用

RuntimeException不需要**显示**的写在方法的末尾，而且可以被**任意**的上层方法栈try...catch...捕获。所以可以在业务应用代码中抛出自定义的RuntimeException，最终统一在拦截器中对异常进行拦截。

如下图所示

```text
拦截器 -> 类1的方法1->类2的方法2-> .....->类3的方法3抛出Runtime异常

拦截器捕获异常 <-类1的方法1捕获异常<-类2的方法2捕获异常<- .....<-类3的方法3抛出Runtime异常
```

