---
title: shiro源码解析-04-权限注解
date: 2021-05-07 09:13:54
tags:
---

## 概述

前不久学习了Shiro 1.4.0提供的权限注解实现权限拦截。最开始猜测实现方式是基于AspectJ方式，即基于`@Apsect`配合`@Pointcut和@After,@AfterReturning,@AfterThrowing,@Around,@Before`实现。具体Spring基于Aspect，AOP实现类可以查看`AnnotationAwareAspectJAutoProxyCreator`。在此提供该类的UML图：

[![Spring基于AspectJ实现的AOP实现类UML图](https://img-blog.csdnimg.cn/20190915155105740.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ExNjkzODg4NDI=,size_16,color_FFFFFF,t_70)](https://img-blog.csdnimg.cn/20190915155105740.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ExNjkzODg4NDI=,size_16,color_FFFFFF,t_70)

[Spring基于AspectJ实现的AOP实现类UML图](https://img-blog.csdnimg.cn/20190915155105740.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ExNjkzODg4NDI=,size_16,color_FFFFFF,t_70)



你可以看到Spring AOP实现继承类大致为ProxyConfig->ProxyProcessorSupport->AbstractAutoProxyCreator

->AbstractAdvisorAutoProxyCreator这条路线。其中Spring常见AOP实现类有3个。

- `DefaultAdvisorAutoProxyCreator`：寻找当前BeanFactory中所有的候选`Advisor(有一个切点和一个通知构成)`，将这些Advisor应用到所有符合切点的Bean中。基于Advisor 匹配规则。
- `BeanNameAutoProxyCreator`：基于Bean配置名规则
- `AnnotationAwareAspectJAutoProxyCreator`：基于Bean中@AspectJ注解匹配规则。

而Shiro权限注解实现原理是基于`DefaultAdvisorAutoProxyCreator`实现的。其权限注解处理Advisor实现类是`AuthorizationAttributeSourceAdvisor`。该类继承了`StaticMethodMatcherPointcutAdvisor`，内部只对Shiro提供的5个权限注解标注的方法或者类进行切面增强。`StaticMethodMatcherPointcutAdvisor`是静态方法切点基类，默认匹配所有的的类。`StaticMethodMatcherPointcut`包括两个主要的子类分别是

`JdkRegexpMethodPointcut`和`NameMatchMethodPointcut`。前者提供正则表达式匹配方法切面，而后者**作为正则表达式的替代，不处理重载方法，默认实现检测xxx\*, \*xxx 和\*xxx\***。

### AuthorizationAttributeSourceAdvisor

```
public class AuthorizationAttributeSourceAdvisor extends StaticMethodMatcherPointcutAdvisor {

    private static final Class<? extends Annotation>[] AUTHZ_ANNOTATION_CLASSES =
            new Class[] {
                    RequiresPermissions.class, RequiresRoles.class,
                    RequiresUser.class, RequiresGuest.class, RequiresAuthentication.class
            };

    protected SecurityManager securityManager = null;

    /**
     * Create a new AuthorizationAttributeSourceAdvisor.
     */
    public AuthorizationAttributeSourceAdvisor() {
        setAdvice(new AopAllianceAnnotationsAuthorizingMethodInterceptor());
    }

    public SecurityManager getSecurityManager() {
        return securityManager;
    }

    public void setSecurityManager(org.apache.shiro.mgt.SecurityManager securityManager) {
        this.securityManager = securityManager;
    }

    public boolean matches(Method method, Class targetClass) {
        Method m = method;

        if ( isAuthzAnnotationPresent(m) ) {
            return true;
        }

        //The 'method' parameter could be from an interface that doesn't have the annotation.
        //Check to see if the implementation has it.
        if ( targetClass != null) {
            try {
                m = targetClass.getMethod(m.getName(), m.getParameterTypes());
                return isAuthzAnnotationPresent(m) || isAuthzAnnotationPresent(targetClass);
            } catch (NoSuchMethodException ignored) {
                //default return value is false.  If we can't find the method, then obviously
                //there is no annotation, so just use the default return value.
            }
        }

        return false;
    }

    // 检测类上是否存在权限验证注解
    private boolean isAuthzAnnotationPresent(Class<?> targetClazz) {
        for( Class<? extends Annotation> annClass : AUTHZ_ANNOTATION_CLASSES ) {
            Annotation a = AnnotationUtils.findAnnotation(targetClazz, annClass);
            if ( a != null ) {
                return true;
            }
        }
        return false;
    }

    // 检测方法上是否存在权限验证注解
    private boolean isAuthzAnnotationPresent(Method method) {
        for( Class<? extends Annotation> annClass : AUTHZ_ANNOTATION_CLASSES ) {
            Annotation a = AnnotationUtils.findAnnotation(method, annClass);
            if ( a != null ) {
                return true;
            }
        }
        return false;
    }

}
```

### Shiro注解权限验证MethodInterceptor

该类是Shiro权限注解拦截器。初始化时，`interceptors`添加了5个方法拦截器，分别对5种权限验证方法进行拦截。

```
public class AopAllianceAnnotationsAuthorizingMethodInterceptor
        extends AnnotationsAuthorizingMethodInterceptor implements MethodInterceptor {

    public AopAllianceAnnotationsAuthorizingMethodInterceptor() {
        List<AuthorizingAnnotationMethodInterceptor> interceptors =
                new ArrayList<AuthorizingAnnotationMethodInterceptor>(5);

        //use a Spring-specific Annotation resolver - Spring's AnnotationUtils is nicer than the
        //raw JDK resolution process.
        AnnotationResolver resolver = new SpringAnnotationResolver();
        //we can re-use the same resolver instance - it does not retain state:
        interceptors.add(new RoleAnnotationMethodInterceptor(resolver));
        interceptors.add(new PermissionAnnotationMethodInterceptor(resolver));
        interceptors.add(new AuthenticatedAnnotationMethodInterceptor(resolver));
        interceptors.add(new UserAnnotationMethodInterceptor(resolver));
        interceptors.add(new GuestAnnotationMethodInterceptor(resolver));

        setMethodInterceptors(interceptors);
    }
    
    ...
    
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        org.apache.shiro.aop.MethodInvocation mi = createMethodInvocation(methodInvocation);
        return super.invoke(mi);
    }
}
```

该类通过调用invoke方法，进而调用超级父类`AuthorizingMethodInterceptor`的invoke方法，在该方法中会先执行`assertAuthorized`方法，进行权限校验，校验不通过抛出`AuthorizationException`。

```
public abstract class AuthorizingMethodInterceptor extends MethodInterceptorSupport {

    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        assertAuthorized(methodInvocation);
        return methodInvocation.proceed();
    }

    protected abstract void assertAuthorized(MethodInvocation methodInvocation) throws AuthorizationException;

}
```

`assertAuthorized`方法最终执行还是`AnnotationsAuthorizingMethodInterceptor`。而`AuthorizingMethodInterceptor`有5个具体的实现类。如下：

- RoleAnnotationMethodInterceptor
- PermissionAnnotationMethodInterceptor
- AuthenticatedAnnotationMethodInterceptor
- UserAnnotationMethodInterceptor
- GuestAnnotationMethodInterceptor

```
public abstract class AnnotationsAuthorizingMethodInterceptor extends AuthorizingMethodInterceptor {
   
    ...
        
    protected void assertAuthorized(MethodInvocation methodInvocation) throws AuthorizationException {
        //default implementation just ensures no deny votes are cast:
        Collection<AuthorizingAnnotationMethodInterceptor> aamis = getMethodInterceptors();
        if (aamis != null && !aamis.isEmpty()) {
            for (AuthorizingAnnotationMethodInterceptor aami : aamis) {
                if (aami.supports(methodInvocation)) {
                    aami.assertAuthorized(methodInvocation);
                }
            }
        }
    }
}
```

`AuthorizingAnnotationMethodInterceptor`首先从子类获取`AuthorizingAnnotationHandler`，再调用该实现类的`assertAuthorized`方法。

```
public abstract class AuthorizingAnnotationMethodInterceptor extends AnnotationMethodInterceptor
{
    
    public AuthorizingAnnotationMethodInterceptor( AuthorizingAnnotationHandler handler ) {
        super(handler);
    }

    public AuthorizingAnnotationMethodInterceptor( AuthorizingAnnotationHandler handler,
                                                   AnnotationResolver resolver) {
        super(handler, resolver);
    }

    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        assertAuthorized(methodInvocation);
        return methodInvocation.proceed();
    }

    public void assertAuthorized(MethodInvocation mi) throws AuthorizationException {
        try {
            ((AuthorizingAnnotationHandler)getHandler()).assertAuthorized(getAnnotation(mi));
        }
        catch(AuthorizationException ae) {
            // Annotation handler doesn't know why it was called, so add the information here if possible. 
            // Don't wrap the exception here since we don't want to mask the specific exception, such as 
            // UnauthenticatedException etc. 
            if (ae.getCause() == null) ae.initCause(new AuthorizationException("Not authorized to invoke method: " + mi.getMethod()));
            throw ae;
        }         
    }
}
```

### RoleAnnotationMethodInterceptor

下面分析一下RoleAnnotationMethodInterceptor。

```
public class RoleAnnotationMethodInterceptor extends AuthorizingAnnotationMethodInterceptor {

    public RoleAnnotationMethodInterceptor() {
        super( new RoleAnnotationHandler() );
    }

	// 具体校验逻辑交给RoleAnnotationHandler
    public RoleAnnotationMethodInterceptor(AnnotationResolver resolver) {
        super(new RoleAnnotationHandler(), resolver);
    }
}
```

RoleAnnotationHandler

```
public class RoleAnnotationHandler extends AuthorizingAnnotationHandler {


    ...
   
    public void assertAuthorized(Annotation a) throws AuthorizationException {
        if (!(a instanceof RequiresRoles)) return;

        RequiresRoles rrAnnotation = (RequiresRoles) a;
        String[] roles = rrAnnotation.value();

        if (roles.length == 1) {
            getSubject().checkRole(roles[0]);
            return;
        }
        if (Logical.AND.equals(rrAnnotation.logical())) {
            getSubject().checkRoles(Arrays.asList(roles));
            return;
        }
        if (Logical.OR.equals(rrAnnotation.logical())) {
            // Avoid processing exceptions unnecessarily - "delay" throwing the exception by calling hasRole first
            boolean hasAtLeastOneRole = false;
            for (String role : roles) if (getSubject().hasRole(role)) hasAtLeastOneRole = true;
            // Cause the exception if none of the role match, note that the exception message will be a bit misleading
            if (!hasAtLeastOneRole) getSubject().checkRole(roles[0]);
        }
    }

}
```

## 实现类似编程式AOP

1. 定义一个注解`LogPrinter`

```
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface LogPrinter {
    String value() default "";
}
```

1. 继承StaticMethodMatcherPointcutAdvisor

```
public class LogPrinterAdvisor extends StaticMethodMatcherPointcutAdvisor {

    public LogPrinterAdvisor() {
        setAdvice(new LogPrinterMethodInterceptor());
    }

    @Override
    public boolean matches(@NonNull Method method, Class<?> targetClass) {
        Method m = method;
        if (isAnnotationPresent(m)) {
            return true;
        }

        if (targetClass != null) {
            try {
                m = targetClass.getMethod(m.getName(), m.getParameterTypes());
                return isAnnotationPresent(m);
            } catch (NoSuchMethodException ignored) {

            }
        }
        return false;
    }

    private boolean isAnnotationPresent(Method method) {
        Annotation a = AnnotationUtils.findAnnotation(method, LogPrinter.class);
        return a != null;
    }
}
```

1. 实现`MethodInterceptor`接口，定义切面处理逻辑

```
public class LogPrinterMethodInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        LogPrinter logPrinter = invocation.getMethod().getAnnotation(LogPrinter.class);

        System.out.println("log printer: "+logPrinter.value());
        return invocation.proceed();
    }
}
```

1. 定义测试类，并添加LogPrinter注解

```
@Component
public class HelloWorld {

    @LogPrinter("hello world")
    public void hello() {
        System.out.println("Hello");
    }
}
```

1. 启动类

```java
@SpringBootApplication
public class LogAdvisorApplication {

    @Bean
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
        defaultAdvisorAutoProxyCreator.setProxyTargetClass(true);
        return defaultAdvisorAutoProxyCreator;
    }

    @Bean
    public LogPrinterAdvisor logPrinterAdvisor() {
        return new LogPrinterAdvisor();
    }

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(LogAdvisorApplication.class, args);
        HelloWorld helloWorld = context.getBean(HelloWorld.class);
        helloWorld.hello();
        context.close();
    }
}
```

## 总结与思考

Shiro的注解式权限，核心配置是一个`DefaultAdvisorAutoProxyCreator`和继承静态方法顾问`StaticMethodMatcherPointcutAdvisor`。其中5种权限注解，使用了统一的代码架构，用到了模板

设计模式。在架构实现上要好于`AspectJ注解实现`。

[![Shiro角色权限注解UML示意图](https://img-blog.csdnimg.cn/20190915155201485.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ExNjkzODg4NDI=,size_16,color_FFFFFF,t_70)](https://img-blog.csdnimg.cn/20190915155201485.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ExNjkzODg4NDI=,size_16,color_FFFFFF,t_70)

[Shiro角色权限注解UML示意图](https://img-blog.csdnimg.cn/20190915155201485.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ExNjkzODg4NDI=,size_16,color_FFFFFF,t_70)



[![Shiro权限注解AOP实现类UML图](https://img-blog.csdnimg.cn/20190915155130196.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ExNjkzODg4NDI=,size_16,color_FFFFFF,t_70)](https://img-blog.csdnimg.cn/20190915155130196.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ExNjkzODg4NDI=,size_16,color_FFFFFF,t_70)

[Shiro权限注解AOP实现类UML图](https://img-blog.csdnimg.cn/20190915155130196.JPG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ExNjkzODg4NDI=,size_16,color_FFFFFF,t_70)



**文章作者:** [知源](mailto:undefined)

**文章链接:** https://crazyblitz.github.io/2019/09/15/Shiro权限注解AOP实现原理分析/

**版权声明:** 本博客所有文章除特别声明外，均采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议。转载请注明来自 [知源博客](https://crazyblitz.github.io/)！