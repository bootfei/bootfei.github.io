---
title: shiro应用-03-会话管理
date: 2021-05-19 17:16:08
tags:
---



所谓会话，即用户访问应用时保持的连接关系，在多次交互中应用能够识别出当前访问的用户是谁，且可以在多次交互中保存一些数据。如访问一些网站时登录成功后，网站可以记住用户，且在退出之前都可以识别当前用户是谁。



Shiro在开启Web功能的时候默认的会话管理器是DefaultWebSessionManager，这种管理器是针对cookie进行存储的，将sessionId存储在cookie中，但是现在的主流方向是前后端分离，我们不能再依赖Cookie，因此我们必须自定义的会话管理器，实现跨JVM，前后端分离。



## **自定义SessionMananger**

- 在原有的DefaultWebSessionManager进行扩展，否则从头实现将会要写大量代码。默认的Web的会话管理器是从cookie中获取SessionId，我们只需要重写其中的方法即可。

```java
/**
 * 自定义的会话管理器
 */
@Slf4j
public class RedisSessionManager extends DefaultWebSessionManager {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 前后端分离不存在cookie，因此需要重写getSessionId的逻辑，从请求参数中获取
     * 此处的逻辑：在登录成功之后会将sessionId作为一个token返回，下次请求的时候直接带着token即可
     */
    @Override
    protected Serializable getSessionId(ServletRequest request, ServletResponse response) {
        //获取上传的token,这里的token就是sessionId
        return request.getParameter("token");
    }


    /**
     * 重写该方法，在SessionManager中只要涉及到Session的操作都会获取Session，获取Session主要是从缓存中获取，父类的该方法执行逻辑如下：
     *  1、先从RedisCache中获取，调用get方法
     *  2、如果RedisCache中不存在，在从SessionDao中获取，调用get方法
     *  优化：我们只需要从SessionDao中获取即可
     * @param sessionKey Session的Key
     */
    @Override
    protected Session retrieveSession(SessionKey sessionKey) throws UnknownSessionException {
        //获取SessionId
        Serializable sessionId = getSessionId(sessionKey);
        if (sessionId == null) {
            log.debug("Unable to resolve session ID from SessionKey [{}].  Returning null to indicate a " +
                    "session could not be found.", sessionKey);
            return null;
        }
        //直接调用SessionDao中的get方法获取
        Session session = ((RedisSessionDao) sessionDAO).doReadSession(sessionId);
        if (session == null) {
            //session ID was provided, meaning one is expected to be found, but we couldn't find one:
            String msg = "Could not find session with ID [" + sessionId + "]";
            throw new UnknownSessionException(msg);
        }
        return session;
    }

    /**
     * 该方法是作用是当访问指定的uri的时候会更新Session中的执行时间，用来动态的延长失效时间。
     * 在父类的实现方法会直接调用SessionDao中的更新方法更新缓存中的Session
     * 此处并没有其他的逻辑，后续可以补充
     * @param key
     */
    @Override
    public void touch(SessionKey key) throws InvalidSessionException {
        super.touch(key);
    }
}

```





## **自定义SessionDao**

- SessionDao的作用是Session持久化的手段，默认的SessionDao是缓存在内存中的，此处使用Redis作为缓存的工具，如下：

```java
/**
 * 自定义RedisSessionDao，继承CachingSessionDAO
 */
@Slf4j
public class RedisSessionDao extends CachingSessionDAO {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 更新session
     * @param session
     */
    @Override
    protected void doUpdate(Session session) {
        log.info("执行redisdao的doUpdate方法");
        redisTemplate.opsForValue().set(session.getId(), session);
    }

    /**
     * 删除session
     * @param session
     */
    @Override
    protected void doDelete(Session session) {
        log.info("执行redisdao的doDelete方法");
        redisTemplate.delete(session.getId());
    }

    /**
     * 创建一个Session，添加到缓存中
     * @param session Session信息
     * @return 创建的SessionId
     */
    @Override
    protected Serializable doCreate(Session session) {
        log.info("执行redisdao的doCreate方法");
        Serializable sessionId = generateSessionId(session);
        assignSessionId(session, sessionId);
        redisTemplate.opsForValue().set(session.getId(), session);
        return sessionId;
    }

    @Override
    protected Session doReadSession(Serializable sessionId) {
        log.info("执行redisdao的doReadSession方法");
       return (Session) redisTemplate.opsForValue().get(sessionId);
    }
}
```



## **自定义SessionId生成策略**

- 默认的Shiro的生成策略是JavaUuidSessionIdGenerator，此处也可以自定义自己的生成策略，如下：

```java
/**
 * 自定义的SessionId的生成策略
 */
public class RedisSessionIdGenerator implements SessionIdGenerator {
    @Override
    public Serializable generateId(Session session) {
        return UUID.randomUUID().toString().replaceAll("-", "").toUpperCase();
    }
}
```



## **自定义Session监听器**

- Session监听器能够监听Session的生命周期，包括开始、过期、失效（停止），如下：

```java
/**
 * 自定义Session监听器
 */
@Slf4j
public class RedisSessionListener implements SessionListener {

    @Override
    public void onStart(Session session) {
        log.info("开始");
    }

    /**
     * Session无效【停止了，stopTime！=null】
     * @param session
     */
    @Override
    public void onStop(Session session) {
        log.info("session失效");
    }

    @Override
    public void onExpiration(Session session) {
        log.info("超时");
    }
}

```





## **完成上述配置**

```java
/**
     * 配置SessionDao，使用自定义的Redis缓存
     */
    @Bean
    public SessionDAO sessionDAO(){
        RedisSessionDao sessionDao = new RedisSessionDao();
        //设置自定义的Id生成策略
        sessionDao.setSessionIdGenerator(new RedisSessionIdGenerator());
        return sessionDao;
    }


    /**
     * 配置会话监听器
     * @return
     */
    @Bean
    public SessionListener sessionListener(){
        return new RedisSessionListener();
    }

    /**
     * 配置会话管理器
     */
    @Bean
    public SessionManager sessionManager(){
        DefaultWebSessionManager sessionManager = new RedisSessionManager();
        //设置session的过期时间
        sessionManager.setGlobalSessionTimeout(60000);
        //设置SessionDao
        sessionManager.setSessionDAO(sessionDAO());
        //设置SessionListener
        sessionManager.setSessionListeners(Lists.newArrayList(sessionListener()));
        return sessionManager;
    }

    /**
     * 配置安全管理器
     */
    @Bean
    public SecurityManager securityManager(){
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager(userRealm());
        //设置缓存管理器
        securityManager.setCacheManager(cacheManager());
        //设置会话管理器
        securityManager.setSessionManager(sessionManager());
        return securityManager;
    }

```





## **优化**

- 在源码中可以看到`AbstractSessionDAO`中的增删改查方法的执行逻辑使用的双层缓存的，还设计到查询CacheManager中的缓存，但是我们的SessionDao既然是实现了Redis的缓存，那么是没必要查询两次的，因此需要重写其中的方法，此时我们自己需要写一个抽象类覆盖其中的增删改查方法即可，如下：

```java
/**
 * RedisSessionDao的抽象类，重写其中的增删改查方法，原因如下：
 *  1、AbstractSessionDAO中的默认方法是写查询CacheManager中的缓存，既然SessionDao实现了Redis的缓存
 *      那么就不需要重复查询两次，因此重写了方法，直接使用RedisSessionDao查询即可。
 */
public abstract class AbstractRedisSessionDao extends AbstractSessionDAO {
    /**
     * 重写creat方法，直接执行sessionDao的方法，不再执行cacheManager
     * @param session
     * @return
     */
    @Override
    public Serializable create(Session session) {
        Serializable sessionId = doCreate(session);
        if (sessionId == null) {
            String msg = "sessionId returned from doCreate implementation is null.  Please verify the implementation.";
            throw new IllegalStateException(msg);
        }
        return sessionId;
    }

    /**
     * 重写删除操作
     * @param session
     */
    @Override
    public void delete(Session session) {
        doDelete(session);
    }

    /**
     * 重写update方法
     * @param session
     * @throws UnknownSessionException
     */
    @Override
    public void update(Session session) throws UnknownSessionException {
        doUpdate(session);
    }

    /**
     * 重写查找方法
     * @param sessionId
     * @return
     * @throws UnknownSessionException
     */
    @Override
    public Session readSession(Serializable sessionId) throws UnknownSessionException {
        Session s = doReadSession(sessionId);
        if (s == null) {
            throw new UnknownSessionException("There is no session with id [" + sessionId + "]");
        }
        return s;
    }

    protected abstract void doDelete(Session session);

    protected abstract void doUpdate(Session session);
}

```



- 此时的上面的RedisSessionDao直接继承我们自定义的抽象类即可，如下：

```java
/**
 * 自定义RedisSessionDao，继承AbstractRedisSessionDao，达到只查一层缓存
 */
@Slf4j
public class RedisSessionDao extends AbstractRedisSessionDao {

    @Autowired
    private RedisTemplate redisTemplate;

    private final static String HASH_NAME="shiro_user";

    /**
     * 更新session
     * @param session
     */
    @Override
    protected void doUpdate(Session session) {
        log.info("执行redisdao的doUpdate方法");
        redisTemplate.opsForHash().put(HASH_NAME, session.getId(), session);
    }

    /**
     * 删除session
     * @param session
     */
    @Override
    protected void doDelete(Session session) {
        log.info("执行redisdao的doDelete方法");
        redisTemplate.opsForHash().delete(HASH_NAME, session.getId());
    }

    /**
     * 创建一个Session，添加到缓存中
     * @param session Session信息
     * @return 创建的SessionId
     */
    @Override
    protected Serializable doCreate(Session session) {
        log.info("执行redisdao的doCreate方法");
        Serializable sessionId = generateSessionId(session);
        assignSessionId(session, sessionId);
        redisTemplate.opsForHash().put(HASH_NAME, session.getId(),session);
        return sessionId;
    }

    @Override
    protected Session doReadSession(Serializable sessionId) {
        log.info("执行redisdao的doReadSession方法");
       return (Session) redisTemplate.opsForHash().get(HASH_NAME,sessionId);
    }


    /**
     * 获取所有的Session
     */
    @Override
    public Collection<Session> getActiveSessions() {
        List values = redisTemplate.opsForHash().values(HASH_NAME);
        if (CollectionUtils.isNotEmpty(values)){
            return values;
        }
        return Collections.emptySet();
    }
}

```

##  会话验证

- Shiro默认是在当前用户访问页面的时候检查Session是否停止或者过期，如果过期和停止了会调用的SessionDao中的相关方法删除缓存，但是如果这是在用户名操作的情况下，如果用户一直未操作，那么Session已经失效了，但是缓存中并没有删除，这样一来将会有大量无效的Session堆积，因此我们必须定时清理失效的Session。

- 清理会话，有如下两种方法：

- - 自己写一个定时器，每隔半小时或者几分钟清除清除缓存
  - 自定义SessionValidationScheduler
  - 使用已经实现的ExecutorServiceSessionValidationScheduler



- 在Shiro中默认会开启ExecutorServiceSessionValidationScheduler，执行时间是一个小时，但是如果想要使用定时器定时清除的话，那么需要关闭默认的清除器，如下：

```java
        //禁用Session清除器，使用定时器清除
        sessionManager.setSessionValidationSchedulerEnabled(false);
```







### **如何的Session是失效的**

- Session是如何保活的？

- - 在`org.apache.shiro.web.servlet.AbstractShiroFilter#doFilterInternal`中的一个`updateSessionLastAccessTime(request, response);`方法用来更新Session的最后执行时间为当前时间，最终调用的就是`org.apache.shiro.session.mgt.SimpleSession#touch`。

  - 在每次请求验证Session的时候实际调用的是`org.apache.shiro.session.mgt.AbstractValidatingSessionManager#doValidate`方法，在其中真正调用的是`org.apache.shiro.session.mgt.SimpleSession#validate`来验证是否过期或者停止

  - - 核心逻辑就是验证当前的时间和最后执行时间的差值是否在设置的过期时间的范围内









### **何时是失效的**

- Session失效目前通过读源码总结出如下三点：

- - isValid判断，这个会在访问请求的时候shiro会自动验证，并且设置进去

  - 用户长期不请求，此时的isValid并不能验证出来，此时需要比较最后执行的时间和开始时间比较

  - 没有登录就访问的也会在redis中生成一个Session，但是此时的Session中是没有两个属性的，以下的两个属性只有在认证成功之后才会设置查到Session中

  - - org.apache.shiro.subject.support.DefaultSubjectContext#PRINCIPALS_SESSION_KEY
    - org.apache.shiro.subject.support.DefaultSubjectContext#AUTHENTICATED_SESSION_KEY





- 通过上面的分析，此时就能写出从缓存中删除失效Session的代码，如下：

```java
/**
     * session过期有三种可能，如下：
     *  1、isValid判断，这个会在访问请求的时候shiro会自动验证，并且设置进去
     *  2、用户长期不请求，此时的isValid并不能验证出来，此时需要比较最后执行的时间和开始时间
     *  3、没有登录就访问的也会在redis中生成一个Session，但是此时的Session中是没有两个属性的
     *      1、org.apache.shiro.subject.support.DefaultSubjectContext#PRINCIPALS_SESSION_KEY
     *      2、org.apache.shiro.subject.support.DefaultSubjectContext#AUTHENTICATED_SESSION_KEY
     */
    public static void clearExpireSession(){
        //获取所有的Session
        Collection<Session> sessions = redisSessionDao.getActiveSessions();
        sessions.forEach(s->{
            SimpleSession session= (SimpleSession) s;
            //第一种可能
            Boolean status1=!session.isValid();
            //第二种可能用开始时间和过期时间比较
            Boolean status2=session.getLastAccessTime().getTime()+session.getTimeout()<new Date().getTime();
            //第三种可能
            Boolean status3= Objects.isNull(session.getAttribute(DefaultSubjectContext.AUTHENTICATED_SESSION_KEY))&&Objects.isNull(session.getAttribute(DefaultSubjectContext.PRINCIPALS_SESSION_KEY));
            if (status1||status2||status3){
                //清楚session
                redisSessionDao.delete(session);
            }
        });
    }
```