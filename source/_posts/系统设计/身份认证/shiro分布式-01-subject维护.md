---
title: shiro分布式-01-subject维护
date: 2021-05-11 11:52:48
tags:
---

## 问题

在技术群有一位提出

   shiro框架下获取主题subject信息都是从SecurityUtils.getSubject()方法中获取，跟踪到最后都是从ThreadLocal变量中获取的值，只能在当前内存中存储，那分布式部署的话，会产生获取到的subject的主体信息不一致的问题？

那么 先看下上述问题的代码：

```csharp
   //SecurityUtils
   public static Subject getSubject() {
        Subject subject = ThreadContext.getSubject();
        if (subject == null) {
            subject = (new Subject.Builder()).buildSubject();
            ThreadContext.bind(subject);
        }
        return subject;
    }
```



```dart
//看下ThreadContext的方法 getSubject->get->getValue->resources
    //1
    public static Subject getSubject() {
        return (Subject) get(SUBJECT_KEY);
    }

//2
   public static Object get(Object key) {
        if (log.isTraceEnabled()) {
            String msg = "get() - in thread [" + Thread.currentThread().getName() + "]";
            log.trace(msg);
        }

        Object value = getValue(key);
        if ((value != null) && log.isTraceEnabled()) {
            String msg = "Retrieved value of type [" + value.getClass().getName() + "] for key [" +
                    key + "] " + "bound to thread [" + Thread.currentThread().getName() + "]";
            log.trace(msg);
        }
        return value;
    }
   //3 
     private static Object getValue(Object key) {
        Map<Object, Object> perThreadResources = resources.get();
        return perThreadResources != null ? perThreadResources.get(key) : null;
    }
    //4
        private static final ThreadLocal<Map<Object, Object>> resources = new InheritableThreadLocalMap<Map<Object, Object>>();
```

单看这些，可能有的朋友也会认同如此，都是从resources这个线程私有变量拿去。

## 分析

### 谁负责从subject取值

根据上述代码分析，是每个请求从subject中获取自己的subject信息

### 谁负责对subject赋值

但是我是做过基于redis分布搭建shiro项目的，肯定应该不会存在上述现象，还是查源码为先吧。

#### 先看这个ThreadLocal变量究竟是哪个类在给他赋值？

```kotlin
//ThreadContext类
//获取的变量key，请注意看，无论怎么取subject，key值永远不变 
    public static final String SECURITY_MANAGER_KEY = ThreadContext.class.getName() + "_SECURITY_MANAGER_KEY";
    public static final String SUBJECT_KEY = ThreadContext.class.getName() + "_SUBJECT_KEY";
    
    //找到一个引用，置入subject的信息
    public static void bind(Subject subject) {
        if (subject != null) {
            put(SUBJECT_KEY, subject);
        }
    }
    
```

对bind方法进行跟踪

```css
--SubjectThreadState.bind()
----SubjectCallable.call()
------DelegatingSubject.execute(Callable<V> callable)
----SubjectRunnable.run()
------DelegatingSubject.execute(Runnable runnable)
```

可以看到SubjectThreadState字面意思应该是线程状态的信息，最终都跟踪到DelegatingSubject这个代理类上，其执行线程方法，更新绑定的subject值；

再继续溯源，找到AbstractShiroFilter类的doFilterInternal。

```java
//
  protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)
            throws ServletException, IOException {

        Throwable t = null;

        try {
            final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
            final ServletResponse response = prepareServletResponse(request, servletResponse, chain);

            final Subject subject = createSubject(request, response);

            //noinspection unchecked
            //在该处调用线程方法更新
            subject.execute(new Callable() {
                public Object call() throws Exception {
                    updateSessionLastAccessTime(request, response);
                    executeChain(request, response, chain);
                    return null;
                }
            });
        } catch (ExecutionException ex) {
            t = ex.getCause();
        } catch (Throwable throwable) {
            t = throwable;
        }
```

看到这里，应该了解到一点，就是每个请求进来的时候，这个主线程会有对应的新线程去更新赋值subject的信息。

#### 每次请求进来时这个subject信息是在这里生成的，那原来已经登录的用户再次来请求的subject已经变了，怎么登录成功呢？还有没登录的时候，这里会怎么处理呢？

借着这个问题，咱么看一下这个shiro过滤器都干啥了？

 一般web项目都指定 ShiroFilter这个过滤器，在初始化到第一个请求到达之前，首先设置了一些环境参数，获取配置信息，初始化一些变量，包含SecurityManager这个核心（可以看init()，onFilterConfigSet()函数）

第一个请求到达进入doFilterInternal方法内，调用prepareServletRequest，prepareServletResponse，判断为http方式时，封装Shiro格式的输入输出流。

还有就是一旦发生请求，那么会话session就已经存在了

接下来创建新的subject具体过程：

```cpp
//AbstractShiroFilter
protected WebSubject createSubject(ServletRequest request, ServletResponse response) {
        return new WebSubject.Builder(getSecurityManager(), request, response).buildWebSubject();
    }
```

跟踪buildWebSubject方法，DefaultSecurityManager类

```cpp
//DefaultSecurityManager
  public Subject createSubject(SubjectContext subjectContext) {
        //create a copy so we don't modify the argument's backing map:
        //英文不好，自行百度吧
        SubjectContext context = copy(subjectContext);

        //ensure that the context has a SecurityManager instance, and if not, add one:
         //英文不好，自行百度吧
        context = ensureSecurityManager(context);

        //Resolve an associated Session (usually based on a referenced session ID), and place it in the context before
        //sending to the SubjectFactory.  The SubjectFactory should not need to know how to acquire sessions as the
        //process is often environment specific - better to shield the SF from these details:
         //这里是重点
        context = resolveSession(context);

        //Similarly, the SubjectFactory should not require any concept of RememberMe - translate that here first
        //if possible before handing off to the SubjectFactory:
        context = resolvePrincipals(context);
        //也是重点
        Subject subject = doCreateSubject(context);

        //save this subject for future reference if necessary:
        //(this is needed here in case rememberMe principals were resolved and they need to be stored in the
        //session, so we don't constantly rehydrate the rememberMe PrincipalCollection on every operation).
        //Added in 1.2:
        
        save(subject);

        return subject;
    }
```

重点跟踪resolveSession()方法



```kotlin
  protected SubjectContext resolveSession(SubjectContext context) {
        //第一次进来肯定没有缓存值，会进入下面的resolveContextSession
        if (context.resolveSession() != null) {
            log.debug("Context already contains a session.  Returning.");
            return context;
        }
        try {
            //Context couldn't resolve it directly, let's see if we can since we have direct access to 
            //the session manager:
            Session session = resolveContextSession(context);
            if (session != null) {
                context.setSession(session);
            }
        } catch (InvalidSessionException e) {
            log.debug("Resolved SubjectContext context session is invalid.  Ignoring and creating an anonymous " +
                    "(session-less) Subject instance.", e);
        }
        return context;
    }
```



```java
   //DefaultSecurityManager
   protected Session resolveContextSession(SubjectContext context) throws InvalidSessionException {
   //这里判断一下是http请求的话，会获取本身session的id
        SessionKey key = getSessionKey(context);
        if (key != null) {
            return getSession(key);
        }
        return null;
    }
```



```java
  //一直跟踪getSession方法到AbstractNativeSessionManager
 public Session getSession(SessionKey key) throws SessionException {
        Session session = lookupSession(key);
        return session != null ? createExposedSession(session, key) : null;
    }
    
     private Session lookupSession(SessionKey key) throws SessionException {
        if (key == null) {
            throw new NullPointerException("SessionKey argument cannot be null.");
        }
        return doGetSession(key);
    }
    
    
```



```java
//上面doGetSession调用了了AbstractValidatingSessionManager
 protected final Session doGetSession(final SessionKey key) throws InvalidSessionException {
        enableSessionValidationIfNecessary();

        log.trace("Attempting to retrieve session with key {}", key);

        Session s = retrieveSession(key);
        if (s != null) {
            validate(s, key);
        }
        return s;
    }
```



```java
//DefaultSessionManager.retrieveSession
 protected Session retrieveSession(SessionKey sessionKey) throws UnknownSessionException {
        Serializable sessionId = getSessionId(sessionKey);
        if (sessionId == null) {
            log.debug("Unable to resolve session ID from SessionKey [{}].  Returning null to indicate a " +
                    "session could not be found.", sessionKey);
            return null;
        }
        Session s = retrieveSessionFromDataSource(sessionId);
        if (s == null) {
            //session ID was provided, meaning one is expected to be found, but we couldn't find one:
            String msg = "Could not find session with ID [" + sessionId + "]";
            throw new UnknownSessionException(msg);
        }
        return s;
    }

//DefaultSessionManager.retrieveSessionFromDataSource
 protected Session retrieveSessionFromDataSource(Serializable sessionId) throws UnknownSessionException {
        return sessionDAO.readSession(sessionId);
    }
```

一直跟踪到sessionDAO.readSession,这里，从方法字面意思也知道是从缓存或者数据库读取session信息了。就是上面提到的redis缓存，会吧session信息存储。

接下来回头看，在buildWebSubject的方法里面，获取到了缓存的session信息或者是一个全新的session信息，

之后的doCreateSubject方法

```java
//DefaultWebSubjectFactory

public Subject createSubject(SubjectContext context) {
        //SHIRO-646
        //Check if the existing subject is NOT a WebSubject. If it isn't, then call super.createSubject instead.
        //Creating a WebSubject from a non-web Subject will cause the ServletRequest and ServletResponse to be null, which wil fail when creating a session.
        boolean isNotBasedOnWebSubject = context.getSubject() != null && !(context.getSubject() instanceof WebSubject);
        if (!(context instanceof WebSubjectContext) || isNotBasedOnWebSubject) {
            return super.createSubject(context);
        }
        WebSubjectContext wsc = (WebSubjectContext) context;
        SecurityManager securityManager = wsc.resolveSecurityManager();
        //取到session信息
        Session session = wsc.resolveSession();
        boolean sessionEnabled = wsc.isSessionCreationEnabled();
        //如果缓存没有，获取session中的身份信息
        PrincipalCollection principals = wsc.resolvePrincipals();
        //如果缓存没有，获取session中的认证信息
        boolean authenticated = wsc.resolveAuthenticated();
        String host = wsc.resolveHost();
        ServletRequest request = wsc.resolveServletRequest();
        ServletResponse response = wsc.resolveServletResponse();
        //创建新的subject
        return new WebDelegatingSubject(principals, authenticated, host, session, sessionEnabled,
                request, response, securityManager);
    }
```

创建完毕以后，接下来就更新缓存内的subject的状态，直至请求结束

到这里每次请求到达filter基本上流程就完毕了，如果已经登录了，无论是从本身的map缓存还是外部的redis缓存，都应该能拿到session信息，如果没有登录，是拿不到的。也证明了分布式部署时，有了session的分布式缓存，应该就基本搭建成功了吧。