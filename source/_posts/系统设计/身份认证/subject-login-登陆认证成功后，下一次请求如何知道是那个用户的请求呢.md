---
title: subject.login()登陆认证成功后，下一次请求如何知道是那个用户的请求呢
date: 2021-04-22 11:38:42
tags:
---



### 背景

总的来说，SecurityUtils.getSubject()是每个请求创建一个Subject, 并保存到ThreadContext的resources（ThreadLocal<Map<Object, Object>>）变量中，也就是一个http请求一个subject,并绑定到当前线程。 

### 问题

问题来了：.subject.login()登陆认证成功后，下一次请求如何知道是那个用户的请求呢？ 

### 原理

首先给出内部原理：1个请求1个Subject原理:由于ShiroFilterFactoryBean本质是个AbstractShiroFilter过滤器，所以每次请求都会执行doFilterInternal里面的createSubject方法。 

#### 猜想：因为subject是绑定到当前线程,这肯定需要一个中介存储状态 

```java
public static Subject getSubject() {  
    Subject subject = ThreadContext.getSubject();  
    if (subject == null) {  
        subject = (new Builder()).buildSubject();  
        ThreadContext.bind(subject);  
   }  
    return subject;  
}  
```

```java
public abstract class ThreadContext {  
    private static final Logger log = LoggerFactory.getLogger(ThreadContext.class);  
    public static final String SECURITY_MANAGER_KEY = ThreadContext.class.getName() + "_SECURITY_MANAGER_KEY";  
    public static final String SUBJECT_KEY = ThreadContext.class.getName() + "_SUBJECT_KEY";  
    private static final ThreadLocal<Map<Object, Object>> resources = new ThreadContext.InheritableThreadLocalMap();  
  
 }
```

#### subject登陆成功后，下一次请求如何知道是那个用户的请求呢？ 

经过源码分析，核心实现如下DefaultSecurityManager类中： 

```java
public Subject createSubject(SubjectContext subjectContext) {  
    SubjectContext context = this.copy(subjectContext);  
    context = this.ensureSecurityManager(context);  
    context = this.resolveSession(context);  
    context = this.resolvePrincipals(context);  
    Subject subject = this.doCreateSubject(context);  
    this.save(subject);  
    return subject;  
}
```

每次请求都会重新设置Session和Principals,看到这里大概就能猜到：如果是web工程，直接从web容器获取httpSession，然后再从httpSession获取Principals，本质就是从cookie获取用户信息，然后每次都设置Principal，这样就知道是哪个用户的请求，并知道这个用户有没有认证成功，--本质：依赖于浏览器的cookie来维护session的 

### 扩展

#### 如果不是web容器的app,如何实现实现无状态的会话 

1.一般的作法会在header中带有一个token，或者是在参数中，后台根据这个token来进行校验这个用户的身份，但是这个时候，servlet中的session就无法保存，我们在这个时候，就要实现自己的会话创建，普通的作法就是重写session与request的接口，然后在过滤器在把它替换成自己的request,所以得到的session也是自己的session，然后根据token来创建和维护会话 

2.shiro实现： 

重写shiro的sessionManage 


```java
public class StatelessSessionManager extends DefaultWebSessionManager {  
    /** 
     * 这个是服务端要返回给客户端， 
     */  
    public final static String TOKEN_NAME = "TOKEN";  
    /** 
     * 这个是客户端请求给服务端带的header 
     */  
    public final static String HEADER_TOKEN_NAME = "token";  
    public final static Logger LOG =         LoggerFactory.getLogger(StatelessSessionManager.class);  
  
    @Override  
    public Serializable getSessionId(SessionKey key) {  
        Serializable sessionId = key.getSessionId();  
        if(sessionId == null){  
            HttpServletRequest request = WebUtils.getHttpRequest(key);  
            HttpServletResponse response = WebUtils.getHttpResponse(key);  
            sessionId = this.getSessionId(request,response);  
        }  
        HttpServletRequest request = WebUtils.getHttpRequest(key);  
        request.setAttribute(TOKEN_NAME,sessionId.toString());  
        return sessionId;  
    }  
  
    @Override  
    protected Serializable getSessionId(ServletRequest servletRequest, ServletResponse servletResponse) {  
        HttpServletRequest request = (HttpServletRequest) servletRequest;  
        String token = request.getHeader(HEADER_TOKEN_NAME);  
        if(token == null){  
            token = UUID.randomUUID().toString();  
        }  
  
        //这段代码还没有去查看其作用，但是这是其父类中所拥有的代码，重写完后我复制了过来...开始  
        request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_SOURCE,  
                ShiroHttpServletRequest.COOKIE_SESSION_ID_SOURCE);  
        request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID, token);  
        request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_IS_VALID, Boolean.TRUE);  
        request.setAttribute(ShiroHttpServletRequest.SESSION_ID_URL_REWRITING_ENABLED, isSessionIdUrlRewritingEnabled());  
        //这段代码还没有去查看其作用，但是这是其父类中所拥有的代码，重写完后我复制了过来...结束  
        return token;  
    }
}
 
```

 登录接口

```

    @RequestMapping("/")  
    public void login(@RequestParam("code")String code, HttpServletRequest request){  
        Map<String,Object> data = new HashMap<>();  
        if(SecurityUtils.getSubject().isAuthenticated()){  
　　　　　　　　//这里代码着已经登陆成功，所以自然不用再次认证，直接从rquest中取出就行了，  
            data.put(StatelessSessionManager.HEADER_TOKEN_NAME,getServerToken());  
            data.put(BIND,ShiroKit.getUser().getTel() != null);  
            response(data);  
        }  
        LOG.info("授权码为:" + code);  
        AuthorizationService authorizationService = authorizationFactory.getAuthorizationService(Constant.clientType);  
        UserDetail authorization = authorizationService.authorization(code);  
  
        Oauth2UserDetail userDetail = (Oauth2UserDetail) authorization;  
  
        loginService.login(userDetail);  
        User user = userService.saveUser(userDetail,Constant.clientType.toString());  
        ShiroKit.getSession().setAttribute(ShiroKit.USER_DETAIL_KEY,userDetail);  
        ShiroKit.getSession().setAttribute(ShiroKit.USER_KEY,user);  
        data.put(BIND,user.getTel() != null);  
　　　　//这里的代码，必须放到login之执行，因为login后，才会创建session，才会得到最新的token咯  
        data.put(StatelessSessionManager.HEADER_TOKEN_NAME,getServerToken());  
        response(data);  
    }

```



```java
@Configuration  
public class ShiroConfiguration {  
  
    @Bean  
    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor(){  
        return new LifecycleBeanPostProcessor();  
    }  
  
    /** 
     * 此处注入一个realm 
     * @param realm 
     * @return 
     */  
    @Bean  
    public SecurityManager securityManager(Realm realm){  
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();  
        securityManager.setSessionManager(new StatelessSessionManager());  
        securityManager.setRealm(realm);  
        return securityManager;  
    }  
  
    @Bean  
    public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager){  
        ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();  
        bean.setSecurityManager(securityManager);  
  
        Map<String,String> map = new LinkedHashMap<>();  
        map.put("/public/**","anon");  
        map.put("/login/**","anon");  
        map.put("/**","user");  
        bean.setFilterChainDefinitionMap(map);  
  
        return bean;  
    }  
}  
```

