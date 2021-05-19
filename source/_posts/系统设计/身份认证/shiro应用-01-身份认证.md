---
title: shiro应用-01-使用流程
date: 2021-05-10 13:25:09
tags:
---

![Image](https://mmbiz.qpic.cn/mmbiz_jpg/JfTPiahTHJhpmuJOMKBHpBCC00gwWgjwZW67mVRo4ffmBpibDMLQicgZJuicvNNnPaiaeXFxOPYY4OXATuhDj6Wnq6Q/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## jdk实现

### 设置Realm域

Shiro要从Realm域中获取安全、正确的数据。该类必须继承org.apache.shiro.realm.AuthorizingRealm；

　　并重写doGetAuthorizationInfo与doGetAuthenticationInfo方法。doGetAuthorizationInfo是对角色权限的认证，这里暂且不详述；doGetAuthenticationInfo对用户的认证，这里正是我们要详细讲述的地方。



```java
public class CustomRealm implements Realm {
    @Override
    public AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        String username = (String)token.getPrincipal();  //得到用户名
        String password = new String((char[])token.getCredentials()); //得到密码
        if(!"zhang".equals(username)) {
            throw new UnknownAccountException(); //如果用户名错误
        }
        if(!"123".equals(password)) {
            throw new IncorrectCredentialsException(); //如果密码错误
        }
        //如果身份认证验证成功，返回一个AuthenticationInfo实现；
        return new SimpleAuthenticationInfo(username, password, getName());
    }
}
```

### 配置文件，使securityManager获取到该Realm域

```bash
1 [main]
2 #自定义 realm
3 customRealm=org.ssm.service.CustomRealm
4 #将realm设置到securityManager
5 securityManager.realms=$customRealm
```

### 测试：main函数

```java
@Test
public void testShiro() {
  //通过工厂加载配置
  Factory<SecurityManager> factory = new IniSecurityManagerFactory("classpath:shiro-shiro.ini");
  //通过工厂获取安全管理器
  SecurityManager securityManager = factory.getInstance();
  //注册Subject工具
  SecurityUtils.setSecurityManager(securityManager);
  //获取Subject对象
  Subject subject = SecurityUtils.getSubject();
  //获取验证对象
  AuthenticationToken authenticationToken = new UsernamePasswordToken("admin", "hello");
  //        验证
  try {
    subject.login(authenticationToken);
    System.out.println("通过..." + "\t" + authenticationToken.getPrincipal());
  } catch (UnknownAccountException e) {
    System.out.println("未通过:用户不存在" + "\t" + authenticationToken.getPrincipal());
  }

}
```



## spring mvc实现

### 配置Shiro的过滤器

对所有访问进行过滤；需要在web.xml中加上如下配置：

```xml
<!--配置上shiro过滤器，使其生效-->
    <filter>
        <filter-name>shiroFilter</filter-name>
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
        <!--设置url由servlet容器控制filter的生命周期-->
        <init-param>
            <param-name>transformWsdlLocations</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>shiroFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
```

### 设置Realm域，且安全数据来源于数据库

```java
public class UserRealm extends AuthorizingRealm {
 
     /**
      * 引入users操作的整合操作
      */
     @Autowired
     @Qualifier("usersService")
     private UsersService usersService;
 
     /**
      *获取授权信息
      * @param principalCollection
      * @return
      */
     @Override
     protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
         return null;
     }
 
     /**
      * 获取获取身份验证相关信息
      * @param authenticationToken
      * @return
      * @throws AuthenticationException
      */
     @Override
     protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
 //        获取用户输入的用户名
         String userName = (String) authenticationToken.getPrincipal();
 //        根据用户名查询该数据
         Users user = usersService.selectByName(userName);
 
         if (user == null){
 //            没有找到账号
             throw new UnknownAccountException();
         }
         SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(user.getUserName(), user.getPassWord(),
 //                此为realm内的方法，用于获取realm的name
                 getName()
         );
         return authenticationInfo;
     }
 }
```

### 配置文件，使securityManager获取到该Realm域，使spring容器获取securityManager

```
<!--shiro的配置-->

    <!--注入自定义的Realm-->
    <bean id="userRealm" class="org.ssm.shiro.realm.UserRealm"/>
    <bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
        <property name="realm" ref="userRealm"/>
    </bean>


    <!--配置ShiroFilter的属性-->
    <bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
        <property name="securityManager" ref="securityManager"/>
        <property name="filterChainDefinitions">
            <value>
                /login = anon
            </value>
        </property>
    </bean>
```



### 测试：拦截器

```java
 public class LoginInterceptor implements HandlerInterceptor {
     @Override
     public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
 
 //        从该回话中获取subject
         Subject subject = SecurityUtils.getSubject();
 //        获取用户信息
         String user = (String) subject.getPrincipal();
 //        如果用户信息为空，则进行拦截
         if (user != null){
             return true;
         }
         response.sendRedirect(request.getContextPath() + "/User/login");
         return false;
     }
 
 
     @Override
     public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
 
     }
 
 
     @Override
     public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
 
     }
 
 
 }
```

