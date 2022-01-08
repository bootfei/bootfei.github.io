---
title: 日志记录-AOP日志切面
date: 2021-06-07 00:07:24
tags:
---

# 切面的使用

## 常见版本切面

- @Aspect => 声明该类为一个注解类

切点注解：

- @Pointcut => 定义一个切点，可以简化代码

通知注解：

- @Before => 在切点之前执行代码
- @After => 在切点之后执行代码
- @AfterReturning => 切点返回内容后执行代码，可以对切点的返回值进行封装
- @AfterThrowing => 切点抛出异常后执行
- @Around => 环绕，在切点前后执行代码

```java
@Component
@Aspect
public class RequestLogAspect {
    private final static Logger LOGGER = LoggerFactory.getLogger(RequestLogAspect.class);

    @Pointcut("execution(* your_package.controller..*(..))")
    public void requestServer() {
    }

    @Around("requestServer()")
    public Object doAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        Object result = proceedingJoinPoint.proceed();
        RequestInfo requestInfo = new RequestInfo();
                requestInfo.setIp(request.getRemoteAddr());
        requestInfo.setUrl(request.getRequestURL().toString());
        requestInfo.setHttpMethod(request.getMethod());
        requestInfo.setClassMethod(String.format("%s.%s", proceedingJoinPoint.getSignature().getDeclaringTypeName(),
                proceedingJoinPoint.getSignature().getName()));
        requestInfo.setRequestParams(getRequestParamsByProceedingJoinPoint(proceedingJoinPoint));
        requestInfo.setResult(result);
        requestInfo.setTimeCost(System.currentTimeMillis() - start);
        LOGGER.info("Request Info      : {}", JSON.toJSONString(requestInfo));

        return result;
    }


    @AfterThrowing(pointcut = "requestServer()", throwing = "e")
    public void doAfterThrow(JoinPoint joinPoint, RuntimeException e) {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        RequestErrorInfo requestErrorInfo = new RequestErrorInfo();
        requestErrorInfo.setIp(request.getRemoteAddr());
        requestErrorInfo.setUrl(request.getRequestURL().toString());
        requestErrorInfo.setHttpMethod(request.getMethod());
        requestErrorInfo.setClassMethod(String.format("%s.%s", joinPoint.getSignature().getDeclaringTypeName(),
                joinPoint.getSignature().getName()));
        requestErrorInfo.setRequestParams(getRequestParamsByJoinPoint(joinPoint));
        requestErrorInfo.setException(e);
        LOGGER.info("Error Request Info      : {}", JSON.toJSONString(requestErrorInfo));
    }

    /**
     * 获取入参
     * @param proceedingJoinPoint
     *
     * @return
     * */
    private Map<String, Object> getRequestParamsByProceedingJoinPoint(ProceedingJoinPoint proceedingJoinPoint) {
        //参数名
        String[] paramNames = ((MethodSignature)proceedingJoinPoint.getSignature()).getParameterNames();
        //参数值
        Object[] paramValues = proceedingJoinPoint.getArgs();

        return buildRequestParam(paramNames, paramValues);
    }

    private Map<String, Object> getRequestParamsByJoinPoint(JoinPoint joinPoint) {
        //参数名
        String[] paramNames = ((MethodSignature)joinPoint.getSignature()).getParameterNames();
        //参数值
        Object[] paramValues = joinPoint.getArgs();

        return buildRequestParam(paramNames, paramValues);
    }

    private Map<String, Object> buildRequestParam(String[] paramNames, Object[] paramValues) {
        Map<String, Object> requestParams = new HashMap<>();
        for (int i = 0; i < paramNames.length; i++) {
            Object value = paramValues[i];

            //如果是文件对象
            if (value instanceof MultipartFile) {
                MultipartFile file = (MultipartFile) value;
                value = file.getOriginalFilename();  //获取文件名
            }

            requestParams.put(paramNames[i], value);
        }

        return requestParams;
    }

    @Data
    public class RequestInfo {
        private String ip;
        private String url;
        private String httpMethod;
        private String classMethod;
        private Object requestParams;
        private Object result;
        private Long timeCost;
    }

    @Data
    public class RequestErrorInfo {
        private String ip;
        private String url;
        private String httpMethod;
        private String classMethod;
        private Object requestParams;
        private RuntimeException exception;
    }
}
```





## 使用自定义注解

### 在pom.xml文件中添加如下依赖

```text
<!-- aop依赖 -->
<dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 修改配置文件

在项目的application.properties文件中添加下面一句配置：

```text
spring.aop.auto=true
```

> 这里特别说明下，这句话不加其实也可以，因为默认就是true，只要我们在pom.xml中添加了依赖就可以了，这里提出来是让大家知道有这个有这个配置。

### 自定义注解

上边介绍过了了，因为这类日志比较灵活，所以我们需要自定义一个注解，使用的时候在需要记录日志的方法上添加这个注解就可以了, 新建new一个名为Log的Annotation文件，文件内容如下：

```java
package com.abc.config;
/**
* @author Promise
* @createTime 2018年12月18日 下午9:26:25
* @description 定义一个方法级别的@log注解
*/
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Log {
 String value() default "";
}
```



### AOP的切面和切点

准备上边的相关文件后，下面来介绍重点--创建AOP切面实现类，同样我们这里将该类放在config包下，命名为LogAsPect.java,内容如下：

```java
package com.abc.config;

@Aspect
@Component
public class LogAsPect {
 

 //表示匹配带有自定义注解的方法
 @Pointcut("@annotation(com.web.springbootaoplog.config.Log)")
 public void pointcut() {}
 
 @Around("pointcut()")
 public Object around(ProceedingJoinPoint point) {
   Object result =null;
   long beginTime = System.currentTimeMillis();

   try {
       log.info("我在目标方法之前执行！");
       result = point.proceed();
       long endTime = System.currentTimeMillis();
       insertLog(point,endTime-beginTime);
   } catch (Throwable e) {
   // TODO Auto-generated catch block
   }
   return result;
 }
 
}
```



### 测试控制器

在controller包下新建一个HomeController

```java
@Controller
public class HomeController {
 private final static Logger log = org.slf4j.LoggerFactory.getLogger(HomeController.class);
     @RequestMapping("/test")
     @ResponseBody
     @Log("测试aoplog")
     public String aop(String name, String nick) {
         log.info("我被执行了！");
					return "123";
     }
}
```

# 添加traceId

## 以log4j2为例

- 添加拦截器

  ```
  public class LogInterceptor implements HandlerInterceptor {
      private final static String TRACE_ID = "traceId";
  
      @Override
      public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
          String traceId = java.util.UUID.randomUUID().toString().replaceAll("-", "").toUpperCase();
          ThreadContext.put("traceId", traceId);
  
          return true;
      }
  
      @Override
      public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
              throws Exception {
      }
  
      @Override
      public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
              throws Exception {
          ThreadContext. remove(TRACE_ID);
      }
  }
  ```

  在调用前通过ThreadContext加入traceId，调用完成后移除

- 修改日志配置文件 在原来的日志格式中 添加traceId的占位符

  ```xml
  <property name="pattern">[TRACEID:%X{traceId}] %d{HH:mm:ss.SSS} %-5level %class{-1}.%M()/%L - %msg%xEx%n</property>
  ```

  

- 执行效果

  ![Image](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfc41ZgumNnAzdYeLT6uwBGooUwDnMZWdDibxib2z2JGQ3gpb5oIPYuyUavPqzN3ebBhmK9daRoKgXjw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



## 以logback为例

DMC是配置logback和log4j使用的，使用方式和ThreadContext差不多，将ThreadContext.put替换为MDC.put即可，同时修改日志配置文件。

log4j2也是可以配合MDC一起使用的

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfc41ZgumNnAzdYeLT6uwBGoUfCJzzRJ9yYzmiaVyp7QSoUiaaUawt2QrhoAnJ1YETsRZ8eiazGhd23kQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

MDC是slf4j包下的，其具体使用哪个日志框架与我们的依赖有关