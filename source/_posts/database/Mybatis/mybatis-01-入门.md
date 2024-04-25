---
title: mybatis-01-入门
date: 2021-04-13 18:33:58
tags:
---





```
第一种写法（1）：

原符号       <        <=      >       >=       &        '        "
替换符号    &lt;    &lt;=   &gt;    &gt;=   &amp;   &apos;  &quot;
例如：sql如下：
create_date_time &gt;= #{startTime} and  create_date_time &lt;= #{endTime}

第二种写法（2）：
大于等于
<![CDATA[ >= ]]>
小于等于
<![CDATA[ <= ]]>
例如：sql如下：
create_date_time <![CDATA[ >= ]]> #{startTime} and  create_date_time <![CDATA[ <= ]]> #{endTime}
————————————————
版权声明：本文为CSDN博主「转角人生」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/xuanzhangran/article/details/60329357
```

