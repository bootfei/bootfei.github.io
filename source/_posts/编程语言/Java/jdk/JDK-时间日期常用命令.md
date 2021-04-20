---
title: jdk时间日期常用命令
date: 2021-04-01 12:17:47
tags:
---



# JAVA

## 获取当前时间

```java

    public static void main(String[] args){
       SimpleDateFormat formatter= new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
			 Date date = new Date(System.currentTimeMillis());
    } 

```

## 时间加减

```java
public static void addDays(Date d, int days){
    Calendar c = Calendar.getInstance();
    c.setTime(d);
    c.add(Calendar.DATE, days);
    d.setTime( c.getTime().getTime() );
}
```

### 时间比较

```java
			Date d1 = sdformat.parse("2019-04-15");
      Date d2 = sdformat.parse("2019-08-10");
      
      if(d1.compareTo(d2) > 0) {
         System.out.println("Date 1 occurs after Date 2");
      } else if(d1.compareTo(d2) < 0) {
         System.out.println("Date 1 occurs before Date 2");
      } else if(d1.compareTo(d2) == 0) {
         System.out.println("Both dates are equal");
      }
```



## string与java.util.Date互转

```
//string->date	
	public static void testStringConvertToDate(){
        String stringDate = "2008-10-05";
        /*yyyy-MM-dd格式一定要与stringDate的格式一致*/
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        try {
            Date date = sdf.parse(stringDate);
        } catch (ParseException e) {           
        }
	}

//date->string    
   public static void testDateConvertToString(){
        Date date = new Date();
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy年MM月dd日");
        String stringDate = sdf.format(date);
   }
    
//将给定的日期格式的字符串转化为想要的格式字符串显示，中间通过Date类型转换
    public static void stringToString(){

        String stringDate = "2008年10月01日10时50分";

        SimpleDateFormat sdf1 = new SimpleDateFormat("yyyy年MM月dd日");
        SimpleDateFormat sdf2 = new SimpleDateFormat("yyyy-MM-dd");
        Date date = null;
        String stringDate2 = null;
        try {
            date = sdf1.parse(stringDate);
            stringDate2 = sdf2.format(date);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

```

## timestamp与java.util.Date互转



## milliseconds与java.util.Date互转

```
String myDate = "2014/10/29 18:10:45";
SimpleDateFormat sdf = new SimpleDateFormat("yyyy/MM/dd HH:mm:ss");
Date date = sdf.parse(myDate);
long millis = date.getTime();
```



## string与java.sql.Date互转

```
java.sql.date.formate(String date)
```









# MYSQL

## MySQL int(10)时间戳转日期

```mysql
>
mysql> SELECT FROM_UNIXTIME(1255033470);
+---------------------------+
| FROM_UNIXTIME(1255033470) |
+---------------------------+
| 2009-10-08 13:24:30       | 
+---------------------------+
1 row in set (0.01 sec)
```



## MYSQL比较日期

DATE_FORMAT入参必须是date类型，不能是字符串类型；比较的对象是字符串

```mysql
DATE_FORMAT(datetime类型/date类型,"%Y-%m-%d") <= "2021-04-31"
```

