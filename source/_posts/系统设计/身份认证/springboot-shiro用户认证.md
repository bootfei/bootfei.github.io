---
title: springboot+shiro用户认证
date: 2021-04-21 10:44:46
tags:
---

## [spring boot 2.0.0 + shiro + redis实现前后端分离的项目](https://www.cnblogs.com/deityjian/p/12485134.html)

本文主要实现shiro的以下几个功能：

​    1.当用户没有登陆时只能访问登录接口，访问其他接口会返回无效的授权码

　　2.当用户登陆成功后，只能访问该用户权限下的接口，其他接口会返回无权限访问

　　3.一个用户不能两个人同时在线，后登录的会自动踢出先登录的用户

本文使用框架如下：

核心框架：spring boot 2.0.0, spring
mvc框架：spring mvc
持久层框架：mybatis
数据库连接池：alibaba druid
安全框架：apache shiro
缓存框架：redis
日志框架：logback
数据库设计：

数据库主要分为5个表，分别是：用户表，角色表，权限表，角色权限表，用户角色表

 ![img](https://img2020.cnblogs.com/i-beta/1645290/202003/1645290-20200313105558942-123715741.png)

 

 

由于数据表结构我直接拷贝的以前的项目，所以上表中很多字段该文不会用到，各位请根据自己的实际情况修改。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
 
    <groupId>com.alex</groupId>
    <artifactId>springboot</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
 
    <name>springboot</name>
    <description>Demo project for Spring Boot</description>
 
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.0.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
 
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <shiro.version>1.4.0</shiro.version>
        <shiro-redis.version>3.1.0</shiro-redis.version>
    </properties>
 
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <!-- 访问静态资源 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
 
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
 
        <!-- 分页插件 -->
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>1.2.5</version>
        </dependency>
        <!-- alibaba的druid数据库连接池 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.0</version>
        </dependency>
 
 
        <!--@Slf4j自动化日志对象-log-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.16</version>
        </dependency>
 
        <!-- shiro spring. -->
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-core</artifactId>
            <version>${shiro.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring</artifactId>
            <version>${shiro.version}</version>
        </dependency>
        <!-- shiro ehcache -->
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-ehcache</artifactId>
            <version>${shiro.version}</version>
        </dependency>
 
        <!-- shiro+redis缓存插件 -->
        <dependency>
            <groupId>org.crazycake</groupId>
            <artifactId>shiro-redis</artifactId>
            <version>${shiro-redis.version}</version>
        </dependency>
 
        <!--工具类-->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.4</version>
        </dependency>
 
        <!-- fastjson阿里巴巴jSON处理器 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.13</version>
        </dependency>
    </dependencies>
 
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
 
</project>
```



编辑application.yml

```
server:
  port: 8080
 
spring:
    datasource:
        name: test
        url: jdbc:mysql://127.0.0.1:3306/springboot
        username: admin
        password: 123456
        # 使用druid数据源
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.jdbc.Driver
        #初始化大小，最小，最大
        initialSize: 5
        minIdle: 5
        maxActive: 20
        # 配置获取连接等待超时的时间
        maxWait: 60000
        # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
        timeBetweenEvictionRunsMillis: 60000
        # 配置一个连接在池中最小生存的时间，单位是毫秒
        minEvictableIdleTimeMillis: 300000
        # 校验SQL，Oracle配置 spring.datasource.validationQuery=SELECT 1 FROM DUAL，如果不配validationQuery项，则下面三项配置无用
        validationQuery: select 'x'
        testWhileIdle: true
        testOnBorrow: false
        testOnReturn: false
        # 打开PSCache，并且指定每个连接上PSCache的大小
        poolPreparedStatements: true
        maxPoolPreparedStatementPerConnectionSize : 20
        # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
        filters: stat, wall, logback
        connectionProperties : druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
    redis:
      host: 127.0.0.1
      port: 6379
      password : 123456
      timeout: 0
      jedis:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0
          max-wait: -1ms
 
 
## 该配置节点为独立的节点，有很多同学容易将这个配置放在spring的节点下，导致配置无法被识别
mybatis:
  mapper-locations: classpath:mapping/*.xml  #注意：一定要对应mapper映射xml文件的所在路径
  type-aliases-package: com.alex.springboot.model  # 注意：对应实体类的路径
#pagehelper分页插件
pagehelper:
    helperDialect: mysql
    reasonable: true
    supportMethodsArguments: true
    params: count=countSql
```



创建ShiroConfig



```
package com.alex.springboot.system.config;
 
import com.alex.springboot.system.shiro.CredentialsMatcher;
import com.alex.springboot.system.shiro.SessionControlFilter;
import com.alex.springboot.system.shiro.SessionManager;
import com.alex.springboot.system.shiro.ShiroRealm;
import org.apache.shiro.mgt.SecurityManager;
import org.apache.shiro.spring.LifecycleBeanPostProcessor;
import org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor;
import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
import org.apache.shiro.web.servlet.SimpleCookie;
import org.crazycake.shiro.RedisCacheManager;
import org.crazycake.shiro.RedisManager;
import org.crazycake.shiro.RedisSessionDAO;
import org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
 
import javax.servlet.Filter;
import java.util.LinkedHashMap;
import java.util.Map;
 
@Configuration
public class ShiroConfig {
 
    @Value("${spring.redis.host}")
    private String redisHost;
 
    @Value("${spring.redis.port}")
    private int redisPort;
 
    @Value("${spring.redis.password}")
    private String redisPassword;
 
    @Bean
    public ShiroFilterFactoryBean shiroFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        // 没有登陆的用户只能访问登陆页面，前后端分离中登录界面跳转应由前端路由控制，后台仅返回json数据
        shiroFilterFactoryBean.setLoginUrl("/common/unauth");
        // 登录成功后要跳转的链接
        //shiroFilterFactoryBean.setSuccessUrl("/auth/index");
        // 未授权界面;
        shiroFilterFactoryBean.setUnauthorizedUrl("common/unauth");
 
        //自定义拦截器
        Map<String, Filter> filtersMap = new LinkedHashMap<String, Filter>();
        //限制同一帐号同时在线的个数。
        filtersMap.put("kickout", kickoutSessionControlFilter());
        shiroFilterFactoryBean.setFilters(filtersMap);
 
        // 权限控制map.
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<String, String>();
        // 公共请求
        filterChainDefinitionMap.put("/common/**", "anon");
        // 静态资源
        filterChainDefinitionMap.put("/static/**", "anon");
        // 登录方法
        filterChainDefinitionMap.put("/admin/login*", "anon"); // 表示可以匿名访问
 
        //此处需要添加一个kickout，上面添加的自定义拦截器才能生效
        filterChainDefinitionMap.put("/admin/**", "authc,kickout");// 表示需要认证才可以访问
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;
    }
 
    @Bean
    public SecurityManager securityManager() {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        // 设置realm.
        securityManager.setRealm(myShiroRealm());
        // 自定义缓存实现 使用redis
        securityManager.setCacheManager(cacheManager());
        // 自定义session管理 使用redis
        securityManager.setSessionManager(sessionManager());
        return securityManager;
    }
 
    /**
     * 身份认证realm; (这个需要自己写，账号密码校验；权限等)
     *
     * @return
     */
    @Bean
    public ShiroRealm myShiroRealm() {
        ShiroRealm myShiroRealm = new ShiroRealm();
        myShiroRealm.setCredentialsMatcher(credentialsMatcher());
        return myShiroRealm;
    }
 
    @Bean
    public CredentialsMatcher credentialsMatcher() {
        return new CredentialsMatcher();
    }
 
    /**
     * cacheManager 缓存 redis实现
     * 使用的是shiro-redis开源插件
     *
     * @return
     */
    public RedisCacheManager cacheManager() {
        RedisCacheManager redisCacheManager = new RedisCacheManager();
        redisCacheManager.setRedisManager(redisManager());
        redisCacheManager.setKeyPrefix("SPRINGBOOT_CACHE:");   //设置前缀
        return redisCacheManager;
    }
 
    /**
     * RedisSessionDAO shiro sessionDao层的实现 通过redis
     * 使用的是shiro-redis开源插件
     */
    @Bean
    public RedisSessionDAO redisSessionDAO() {
        RedisSessionDAO redisSessionDAO = new RedisSessionDAO();
        redisSessionDAO.setRedisManager(redisManager());
        redisSessionDAO.setKeyPrefix("SPRINGBOOT_SESSION:");
        return redisSessionDAO;
    }
 
    /**
     * Session Manager
     * 使用的是shiro-redis开源插件
     */
    @Bean
    public SessionManager sessionManager() {
        SimpleCookie simpleCookie = new SimpleCookie("Token");
        simpleCookie.setPath("/");
        simpleCookie.setHttpOnly(false);
 
        SessionManager sessionManager = new SessionManager();
        sessionManager.setSessionDAO(redisSessionDAO());
        sessionManager.setSessionIdCookieEnabled(false);
        sessionManager.setSessionIdUrlRewritingEnabled(false);
        sessionManager.setDeleteInvalidSessions(true);
        sessionManager.setSessionIdCookie(simpleCookie);
        return sessionManager;
    }
 
 
    /**
     * 配置shiro redisManager
     * 使用的是shiro-redis开源插件
     *
     * @return
     */
    public RedisManager redisManager() {
        RedisManager redisManager = new RedisManager();
        redisManager.setHost(redisHost);
        redisManager.setPort(redisPort);
        redisManager.setTimeout(1800); //设置过期时间
        redisManager.setPassword(redisPassword);
        return redisManager;
    }
 
    /**
     * 限制同一账号登录同时登录人数控制
     *
     * @return
     */
    @Bean
    public SessionControlFilter kickoutSessionControlFilter() {
        SessionControlFilter kickoutSessionControlFilter = new SessionControlFilter();
        kickoutSessionControlFilter.setCache(cacheManager());
        kickoutSessionControlFilter.setSessionManager(sessionManager());
        kickoutSessionControlFilter.setKickoutAfter(false);
        kickoutSessionControlFilter.setMaxSession(1);
        kickoutSessionControlFilter.setKickoutUrl("/common/kickout");
        return kickoutSessionControlFilter;
    }
 
 
    /***
     * 授权所用配置
     *
     * @return
     */
    @Bean
    public DefaultAdvisorAutoProxyCreator getDefaultAdvisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
        defaultAdvisorAutoProxyCreator.setProxyTargetClass(true);
        return defaultAdvisorAutoProxyCreator;
    }
 
    /***
     * 使授权注解起作用不如不想配置可以在pom文件中加入
     * <dependency>
     *<groupId>org.springframework.boot</groupId>
     *<artifactId>spring-boot-starter-aop</artifactId>
     *</dependency>
     * @param securityManager
     * @return
     */
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager){
        AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
        authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
        return authorizationAttributeSourceAdvisor;
    }
 
    /**
     * Shiro生命周期处理器
     * 此方法需要用static作为修饰词，否则无法通过@Value()注解的方式获取配置文件的值
     *
     */
    @Bean
    public static LifecycleBeanPostProcessor getLifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

自定义Realm

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.alex.springboot.system.shiro;
 
import com.alex.springboot.model.Menu;
import com.alex.springboot.model.Role;
import com.alex.springboot.model.User;
import com.alex.springboot.service.MenuService;
import com.alex.springboot.service.RoleService;
import com.alex.springboot.service.UserService;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections.CollectionUtils;
import org.apache.commons.lang3.StringUtils;
import org.apache.shiro.authc.*;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.subject.PrincipalCollection;
import org.springframework.beans.factory.annotation.Autowired;
 
import java.util.List;
 
@Slf4j
public class ShiroRealm extends AuthorizingRealm {
    @Autowired
    private UserService userService;
    @Autowired
    private RoleService roleService;
    @Autowired
    private MenuService menuService;
    /**
     * 认证信息.(身份验证) : Authentication 是用来验证用户身份
     *
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authcToken) throws AuthenticationException {
        log.info("---------------- 执行 Shiro 凭证认证 ----------------------");
        UsernamePasswordToken token = (UsernamePasswordToken) authcToken;
        String name = token.getUsername();
        // 从数据库获取对应用户名密码的用户
        User user = userService.getUserByName(name);
        if (user != null) {
            // 用户为禁用状态
            if (!user.getLoginFlag().equals("1")) {
                throw new DisabledAccountException();
            }
            log.info("---------------- Shiro 凭证认证成功 ----------------------");
            SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(
                    user, //用户
                    user.getPassword(), //密码
                    getName()  //realm name
            );
            return authenticationInfo;
        }
        throw new UnknownAccountException();
    }
 
    /**
     * 授权
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        log.info("---------------- 执行 Shiro 权限获取 ---------------------");
        Object principal = principals.getPrimaryPrincipal();
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        if (principal instanceof User) {
            User userLogin = (User) principal;
            if(userLogin != null){
                List<Role> roleList = roleService.findByUserid(userLogin.getId());
                if(CollectionUtils.isNotEmpty(roleList)){
                    for(Role role : roleList){
                        info.addRole(role.getEnname());
 
                        List<Menu> menuList = menuService.getAllMenuByRoleId(role.getId());
                        if(CollectionUtils.isNotEmpty(menuList)){
                            for (Menu menu : menuList){
                                if(StringUtils.isNoneBlank(menu.getPermission())){
                                    info.addStringPermission(menu.getPermission());
                                }
                            }
                        }
                    }
                }
            }
        }
        log.info("---------------- 获取到以下权限 ----------------");
        log.info(info.getStringPermissions().toString());
        log.info("---------------- Shiro 权限获取成功 ----------------------");
        return info;
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

自定义密码校验器

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.alex.springboot.system.shiro;
 
import com.alex.springboot.utils.MD5Util;
import org.apache.shiro.authc.AuthenticationInfo;
import org.apache.shiro.authc.AuthenticationToken;
import org.apache.shiro.authc.UsernamePasswordToken;
import org.apache.shiro.authc.credential.SimpleCredentialsMatcher;
 
public class CredentialsMatcher extends SimpleCredentialsMatcher {
 
    @Override
    public boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) {
        UsernamePasswordToken utoken = (UsernamePasswordToken) token;
        // 获得用户输入的密码:(可以采用加盐(salt)的方式去检验)
        String inPassword = new String(utoken.getPassword());
        // 获得数据库中的密码
        String dbPassword = (String) info.getCredentials();
        // 进行密码的比对
        return this.equals(MD5Util.encrypt(inPassword), dbPassword);
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

自定义session容器，用于实现前后端分离，前端请求接口时将Token放在请求Header中，即可获取到用户的session信息（建议前端是将Token放在Header中，而不是放到body请求参数中，这样可以统一做封装处理，下面的代码中是获取Header中或者body中Token，建议直接获取Header中Token即可）

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.alex.springboot.system.shiro;
 
import org.apache.commons.lang3.StringUtils;
import org.apache.shiro.session.Session;
import org.apache.shiro.session.UnknownSessionException;
import org.apache.shiro.session.mgt.SessionKey;
import org.apache.shiro.web.servlet.ShiroHttpServletRequest;
import org.apache.shiro.web.session.mgt.DefaultWebSessionManager;
import org.apache.shiro.web.session.mgt.WebSessionKey;
import org.apache.shiro.web.util.WebUtils;
 
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import java.io.Serializable;
 
public class SessionManager extends DefaultWebSessionManager {
    private static final String AUTHORIZATION = "Token";
 
    private static final String REFERENCED_SESSION_ID_SOURCE = "Stateless request";
 
    public SessionManager() {
    }
 
    @Override
    protected Serializable getSessionId(ServletRequest request, ServletResponse response) {
        //获取请求头，或者请求参数中的Token
        String id = StringUtils.isEmpty(WebUtils.toHttp(request).getHeader(AUTHORIZATION))
                ? request.getParameter(AUTHORIZATION) : WebUtils.toHttp(request).getHeader(AUTHORIZATION);
        // 如果请求头中有 Token 则其值为sessionId
        if (StringUtils.isNotEmpty(id)) {
            request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_SOURCE, REFERENCED_SESSION_ID_SOURCE);
            request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID, id);
            request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_IS_VALID, Boolean.TRUE);
 
            return id;
        } else {
            // 否则按默认规则从cookie取sessionId
            return super.getSessionId(request, response);
        }
    }
 
    /**
     * 获取session 优化单次请求需要多次访问redis的问题
     *
     * @param sessionKey
     * @return
     * @throws UnknownSessionException
     */
    @Override
    protected Session retrieveSession(SessionKey sessionKey) throws UnknownSessionException {
        Serializable sessionId = getSessionId(sessionKey);
 
        ServletRequest request = null;
        if (sessionKey instanceof WebSessionKey) {
            request = ((WebSessionKey) sessionKey).getServletRequest();
        }
 
        if (request != null && null != sessionId) {
            Object sessionObj = request.getAttribute(sessionId.toString());
            if (sessionObj != null) {
                return (Session) sessionObj;
            }
        }
 
        Session session = super.retrieveSession(sessionKey);
        if (request != null && null != sessionId) {
            request.setAttribute(sessionId.toString(), session);
        }
        return session;
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

自定义拦截器，用于限制用户登录人数

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.alex.springboot.system.shiro;
 
import com.alex.springboot.model.User;
import com.alibaba.fastjson.JSON;
import org.apache.shiro.cache.Cache;
import org.apache.shiro.cache.CacheManager;
import org.apache.shiro.session.Session;
import org.apache.shiro.session.mgt.DefaultSessionKey;
import org.apache.shiro.session.mgt.SessionManager;
import org.apache.shiro.subject.Subject;
import org.apache.shiro.web.filter.AccessControlFilter;
import org.apache.shiro.web.util.WebUtils;
 
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.io.PrintWriter;
import java.io.Serializable;
import java.util.Deque;
import java.util.HashMap;
import java.util.LinkedList;
import java.util.Map;
 
 
public class SessionControlFilter extends AccessControlFilter {
    private String kickoutUrl; //踢出后到的地址
    private boolean kickoutAfter = false; //踢出之前登录的/之后登录的用户 默认踢出之前登录的用户
    private int maxSession = 1; //同一个帐号最大会话数 默认1
 
    private SessionManager sessionManager;
    private Cache<String, Deque<Serializable>> cache;
 
    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        return false;
    }
 
    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        Subject subject = getSubject(request, response);
        if(!subject.isAuthenticated() && !subject.isRemembered()) {
            //如果没有登录，直接进行之后的流程
            return true;
        }
 
 
        Session session = subject.getSession();
        User user = (User) subject.getPrincipal();
        String username = user.getLoginName();
        Serializable sessionId = session.getId();
 
        //读取缓存   没有就存入
        Deque<Serializable> deque = cache.get(username);
 
        //如果此用户没有session队列，也就是还没有登录过，缓存中没有
        //就new一个空队列，不然deque对象为空，会报空指针
        if(deque == null){
            deque = new LinkedList<Serializable>();
        }
 
        //如果队列里没有此sessionId，且用户没有被踢出；放入队列
        if(!deque.contains(sessionId) && session.getAttribute("kickout") == null) {
            //将sessionId存入队列
            deque.push(sessionId);
            //将用户的sessionId队列缓存
            cache.put(username, deque);
        }
 
        //如果队列里的sessionId数超出最大会话数，开始踢人
        while(deque.size() > maxSession) {
            Serializable kickoutSessionId = null;
            if(kickoutAfter) { //如果踢出后者
                kickoutSessionId = deque.removeFirst();
                //踢出后再更新下缓存队列
                cache.put(username, deque);
            } else { //否则踢出前者
                kickoutSessionId = deque.removeLast();
                //踢出后再更新下缓存队列
                cache.put(username, deque);
            }
 
            try {
                //获取被踢出的sessionId的session对象
                Session kickoutSession = sessionManager.getSession(new DefaultSessionKey(kickoutSessionId));
                if(kickoutSession != null) {
                    //设置会话的kickout属性表示踢出了
                    kickoutSession.setAttribute("kickout", true);
                }
            } catch (Exception e) {//ignore exception
 
            }
        }
 
        //如果被踢出了，直接退出，重定向到踢出后的地址
        if (session.getAttribute("kickout") != null) {
            //会话被踢出了
            try {
                //退出登录
                subject.logout();
            } catch (Exception e) { //ignore
            }
            saveRequest(request);
 
            Map<String, String> resultMap = new HashMap<String, String>();
            //判断是不是Ajax请求
            if ("XMLHttpRequest".equalsIgnoreCase(((HttpServletRequest) request).getHeader("X-Requested-With"))) {
                resultMap.put("user_status", "300");
                resultMap.put("message", "您已经在其他地方登录，请重新登录！");
                //输出json串
                out(response, resultMap);
            }else{
                //重定向
                WebUtils.issueRedirect(request, response, kickoutUrl);
            }
            return false;
        }
        return true;
    }
 
    private void out(ServletResponse hresponse, Map<String, String> resultMap)
            throws IOException {
        try {
            hresponse.setCharacterEncoding("UTF-8");
            PrintWriter out = hresponse.getWriter();
            out.println(JSON.toJSONString(resultMap));
            out.flush();
            out.close();
        } catch (Exception e) {
            System.err.println("KickoutSessionFilter.class 输出JSON异常，可以忽略。");
        }
    }
 
    public String getKickoutUrl() {
        return kickoutUrl;
    }
 
    public void setKickoutUrl(String kickoutUrl) {
        this.kickoutUrl = kickoutUrl;
    }
 
    public boolean isKickoutAfter() {
        return kickoutAfter;
    }
 
    public void setKickoutAfter(boolean kickoutAfter) {
        this.kickoutAfter = kickoutAfter;
    }
 
    public int getMaxSession() {
        return maxSession;
    }
 
    public void setMaxSession(int maxSession) {
        this.maxSession = maxSession;
    }
 
    public SessionManager getSessionManager() {
        return sessionManager;
    }
 
    public void setSessionManager(SessionManager sessionManager) {
        this.sessionManager = sessionManager;
    }
 
    public Cache<String, Deque<Serializable>> getCache() {
        return cache;
    }
 
    public void setCache(CacheManager cacheManager) {
        this.cache = cacheManager.getCache("shiro_redis_cache");
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

全局异常拦截

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.alex.springboot.system.handler;
 
import com.alex.springboot.system.enums.ResultStatusCode;
import com.alex.springboot.system.vo.Result;
import lombok.extern.slf4j.Slf4j;
import org.apache.shiro.authz.UnauthorizedException;
import org.springframework.http.HttpStatus;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.validation.BindException;
import org.springframework.web.HttpRequestMethodNotSupportedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.MissingServletRequestParameterException;
import org.springframework.web.bind.ServletRequestBindingException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.ResponseStatus;
 
import javax.validation.ConstraintViolationException;
 
@Slf4j
@ControllerAdvice
@ResponseBody
public class ExceptionAdvice {
    /**
     * 400 - Bad Request
     */
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler({HttpMessageNotReadableException.class, MissingServletRequestParameterException.class, BindException.class,
            ServletRequestBindingException.class, MethodArgumentNotValidException.class, ConstraintViolationException.class})
    public Result handleHttpMessageNotReadableException(Exception e) {
        log.error("参数解析失败", e);
        if (e instanceof BindException){
            return new Result(ResultStatusCode.BAD_REQUEST.getCode(), ((BindException)e).getAllErrors().get(0).getDefaultMessage());
        }
        return new Result(ResultStatusCode.BAD_REQUEST.getCode(), e.getMessage());
    }
 
    /**
     * 405 - Method Not Allowed
     */
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public Result handleHttpRequestMethodNotSupportedException(HttpRequestMethodNotSupportedException e) {
        log.error("不支持当前请求方法", e);
        return new Result(ResultStatusCode.METHOD_NOT_ALLOWED, null);
    }
 
    /**
     * shiro权限异常处理
     * @return
     */
    @ExceptionHandler(UnauthorizedException.class)
    public Result unauthorizedException(UnauthorizedException e){
        log.error(e.getMessage(), e);
 
        return new Result(ResultStatusCode.UNAUTHO_ERROR);
    }
 
    /**
     * 500
     * @param e
     * @return
     */
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {
        e.printStackTrace();
        log.error("服务运行异常", e);
        return new Result(ResultStatusCode.SYSTEM_ERR, null);
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

未授权和被踢出后跳转方法

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.alex.springboot.controller;
 
import com.alex.springboot.system.enums.ResultStatusCode;
import com.alex.springboot.system.vo.Result;
import org.apache.shiro.SecurityUtils;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
 
@RequestMapping("/common")
@RestController
public class CommonController {
 
    /**
     * 未授权跳转方法
     * @return
     */
    @RequestMapping("/unauth")
    public Result unauth(){
        SecurityUtils.getSubject().logout();
        return new Result(ResultStatusCode.UNAUTHO_ERROR);
    }
 
    /**
     * 被踢出后跳转方法
     * @return
     */
    @RequestMapping("/kickout")
    public Result kickout(){
        return new Result(ResultStatusCode.INVALID_TOKEN);
    }
 
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

登录和退出登录

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.alex.springboot.controller;
 
import com.alex.springboot.system.enums.ResultStatusCode;
import com.alex.springboot.system.vo.Result;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authc.IncorrectCredentialsException;
import org.apache.shiro.authc.LockedAccountException;
import org.apache.shiro.authc.UnknownAccountException;
import org.apache.shiro.authc.UsernamePasswordToken;
import org.apache.shiro.subject.Subject;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
 
@RestController
@RequestMapping("/admin")
public class LoginController {
 
    @RequestMapping("/login")
    public Result login(String loginName, String pwd){
        try {
            UsernamePasswordToken token = new UsernamePasswordToken(loginName, pwd);
            //登录不在该处处理，交由shiro处理
            Subject subject = SecurityUtils.getSubject();
            subject.login(token);
 
            if (subject.isAuthenticated()) {
                JSON json = new JSONObject();
                ((JSONObject) json).put("token", subject.getSession().getId());
 
                return new Result(ResultStatusCode.OK, json);
            }else{
                return new Result(ResultStatusCode.SHIRO_ERROR);
            }
        }catch (IncorrectCredentialsException | UnknownAccountException e){
            return new Result(ResultStatusCode.NOT_EXIST_USER_OR_ERROR_PWD);
        }catch (LockedAccountException e){
            return new Result(ResultStatusCode.USER_FROZEN);
        }catch (Exception e){
            return new Result(ResultStatusCode.SYSTEM_ERR);
        }
    }
 
    /**
     * 退出登录
     * @return
     */
    @RequestMapping("/logout")
    public Result logout(){
        SecurityUtils.getSubject().logout();
        return new Result(ResultStatusCode.OK);
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

测试接口方法

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
package com.alex.springboot.controller;
 
import com.alex.springboot.service.UserService;
import com.alex.springboot.system.vo.Grid;
import org.apache.shiro.authz.annotation.RequiresPermissions;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
 
@RestController
@RequestMapping("/admin/user")
public class UserController {
 
    @Autowired
    private UserService userService;
 
    @RequiresPermissions("sys:user:view")
    @RequestMapping("findList")
    public Grid findList(){
        return userService.findList();
    }
 
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

![img](https://img2020.cnblogs.com/i-beta/1645290/202003/1645290-20200313110633859-1021094929.png)

 

 Demo地址：https://github.com/DeityJian/springboot-shiro.git