---
title: shiro应用-05-过滤器
date: 2021-05-24 17:39:13
tags:
---

我们都知道shiro是个认证权限框架，除了登录、退出逻辑我们需要侵入项目代码之外，验证用户是否已经登录、是否拥有权限的代码其实都是过滤器来完成的，可以这么说，shiro其实就是一个过滤器链集合。

那么今天我们详细讨论一下shiro底层到底给我们提供了多少默认的过滤器供我们使用，又都有什么用呢？带着问题，我们先去shiro官网看看对于默认过滤器集的说明。

- [http://shiro.apache.org/web.h...](http://shiro.apache.org/web.html#default-filters)

> When running a web-app, Shiro will create some useful default Filter instances and make them available in the [main] section automatically. You can configure them in main as you would any other bean and reference them in your chain definitions.
>
> The default Filter instances available automatically are defined by the DefaultFilter enum and the enum’s name field is the name available for configuration.

默认筛选器实例由DefaultFilter enum中定义，enum s name字段是可用于配置的名称。

```
public enum DefaultFilter {

    anon(AnonymousFilter.class),
    authc(FormAuthenticationFilter.class),
    authcBasic(BasicHttpAuthenticationFilter.class),
    logout(LogoutFilter.class),
    noSessionCreation(NoSessionCreationFilter.class),
    perms(PermissionsAuthorizationFilter.class),
    port(PortFilter.class),
    rest(HttpMethodPermissionFilter.class),
    roles(RolesAuthorizationFilter.class),
    ssl(SslFilter.class),
    user(UserFilter.class);

    ...
}
```

终于知道我们常用的anon、authc、perms、roles、user过滤器是哪里来的了！这些过滤器我们都是可以直接使用的。

<img src="https://image-1300566513.cos.ap-guangzhou.myqcloud.com/upload/images/20200511/8e77bde372b64a83baa2c7d782137a61.png" alt="img" style="zoom: 33%;" />



#### AbstractFilter

这个过滤器还得说说，shiro最底层的抽象过滤器，虽然我们极少直接继承它，它通过实现`Filter`获得过滤器的特性。

完成一些过滤器基本初始化操作，`FilterConfig`：过滤器配置对象，用于servlet容器在初始化期间将信息传递给其他过滤器。

#### NameableFilter

命名过滤器，给过滤器定义名称！也是比较基层的过滤器了，未拓展其他功能，我们很少会直接继承这个过滤器。为重写doFilter方法。

#### OncePerRequestFilter

重写doFilter方法，保证每个servlet方法只会被过滤一次。可以看到doFilter方法中，第一行代码就是`String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();`然后通过`request.getAttribute(alreadyFilteredAttributeName) != null`来判断过滤器是否已经被调用过，从而保证过滤器不会被重复调用。

进入方法之前，先标记`alreadyFilteredAttributeName`为True，抽象`doFilterInternal`方法执行之后再remove掉`alreadyFilteredAttributeName`。

![img](https://image-1300566513.cos.ap-guangzhou.myqcloud.com/upload/images/20200511/391f6d511eb5486e82611a2774eac82f.png)

所以OncePerRequestFilter过滤器保证只会被一次调用的功能，提供了抽象方法`doFilterInternal`让后面的过滤器可以重写，执行真正的过滤器处理逻辑。

```
protected abstract void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain)
            throws ServletException, IOException;
```



#### AdviceFilter

看到Advice，很自然想到切面环绕编程，一般有pre、post、after几个方法。所以这个AdviceFilter过滤器就是提供了和AOP相似的切面功能。

继承OncePerRequestFilter过滤器重写doFilterInternal方法，我们可以先看看：

![img](https://image-1300566513.cos.ap-guangzhou.myqcloud.com/upload/images/20200511/1f169663547f42119c98d93f4d4ab97b.png)

可以看到上面4个序号：

1. preHandle 前置过滤，默认true
2. executeChain 执行真正代码过滤逻辑->chain.doFilter
3. postHandle 后置过滤
4. cleanup 其实主要逻辑是afterCompletion方法

于是，我们从OncePerRequestFilter的一个doFilterInternal分化成了切面编程，更容易前后控制执行逻辑。所以如果继承AdviceFilter时候，我们可以重写preHandle方法，判断用户是否满足已登录或者其他业务逻辑，返回false时候表示不通过过滤器。

#### PathMatchingFilter

请求路径匹配过滤器，通过匹配请求url，判断请求是否需要过滤，如果url未在需要过滤的集合内，则跳过，否则进入`isFilterChainContinued`的onPreHandle方法。

我们可以看下代码：

![img](https://image-1300566513.cos.ap-guangzhou.myqcloud.com/upload/images/20200511/62f7317fd4ec467682f3da3aa11a3c87.png)

![img](https://image-1300566513.cos.ap-guangzhou.myqcloud.com/upload/images/20200511/b568d5dceb2b4a19aad6fd72348f5e58.png)

从上面3个步骤中可以看到，PathMatchingFilter提供的功能是：自定义匹配url，匹配上的请求最终跳转到`onPreHandle`方法。

这个过滤器为后面的常用过滤器提供的基础，比如我们在config中配置如下

```
/login = anon
/admin/* = authc
```

拦截/login请求，经过AnonymousFilter过滤器，我们可以看下

- org.apache.shiro.web.filter.authc.AnonymousFilter

```
public class AnonymousFilter extends PathMatchingFilter {

    /**
     * 公众号：MarkerHub
    **/
    @Override
    protected boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) {
        // Always return true since we allow access to anyone
        return true;
    }
}
```

AnonymousFilter重写了onPreHandle方法，只不过直接返回了true，说明拦截的链接可以直接通过，不需要其他拦截逻辑。

而authc->FormAuthenticationFilter也是间接继承了PathMatchingFilter。

```
public class FormAuthenticationFilter extends AuthenticatingFilter
```

所以，需要拦截某个链接进行业务逻辑过滤的可以继承PathMatchingFilter方法拓展哈。

#### AccessControlFilter

访问控制过滤器。继承PathMatchingFilter过滤器，重写onPreHandle方法，又分出了两个抽象方法来控制

![img](https://image-1300566513.cos.ap-guangzhou.myqcloud.com/upload/images/20200511/63cb9b7e06f249089c88fa25f480ddd3.png)

- isAccessAllowed 是否允许访问
- onAccessDenied 是否拒绝访问

所以，我们现在可以通过重写这个抽象两个方法来控制过滤逻辑。另外多提供了3个方法，方便后面的过滤器使用。

```java
protected void saveRequestAndRedirectToLogin(ServletRequest request, ServletResponse response) throws IOException {
    saveRequest(request);
    redirectToLogin(request, response);
}

protected void saveRequest(ServletRequest request) {
    WebUtils.saveRequest(request);
}

protected void redirectToLogin(ServletRequest request, ServletResponse response) throws IOException {
    String loginUrl = getLoginUrl();
    WebUtils.issueRedirect(request, response, loginUrl);
}
```

其中redirectToLogin提供了调整到登录页面的逻辑与实现，为后面的过滤器发现未登录跳转到登录页面提供了基础。

#### AuthenticationFilter

继承AccessControlFilter，重写了isAccessAllowed方法，通过判断用户是否已经完成登录来判断用户是否允许继续后面的逻辑判断。这里可以看出，从这个过滤器开始，后续的判断会与用户的登录状态相关，直接继承这些过滤器，我们不需要再自己手动去判断用户是否已经登录。并且提供了登录成功之后跳转的方法。

```
public abstract class AuthenticationFilter extends AccessControlFilter {
    public void setSuccessUrl(String successUrl) {
        this.successUrl = successUrl;
    }

    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        Subject subject = getSubject(request, response);
        return subject.isAuthenticated();
    }
}
```

#### AuthenticatingFilter

继承AuthenticationFilter，提供了自动登录、是否登录请求等方法。

```java
/**
 * 公众号：MarkerHub
**/
public abstract class AuthenticatingFilter extends AuthenticationFilter {
    public static final String PERMISSIVE = "permissive";

    //TODO - complete JavaDoc

    protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
        AuthenticationToken token = createToken(request, response);
        if (token == null) {
            String msg = "createToken method implementation returned null. A valid non-null AuthenticationToken " +
                    "must be created in order to execute a login attempt.";
            throw new IllegalStateException(msg);
        }
        try {
            Subject subject = getSubject(request, response);
            subject.login(token);
            return onLoginSuccess(token, subject, request, response);
        } catch (AuthenticationException e) {
            return onLoginFailure(token, e, request, response);
        }
    }

    protected abstract AuthenticationToken createToken(ServletRequest request, ServletResponse response) throws Exception;

    /**
     * 公众号：MarkerHub
    **/
    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        return super.isAccessAllowed(request, response, mappedValue) ||
                (!isLoginRequest(request, response) && isPermissive(mappedValue));
    }
    ...
}
```

- executeLogin 执行登录
- onLoginSuccess 登录成功跳转
- onLoginFailure 登录失败跳转
- createToken 创建登录的身份token
- isAccessAllowed 是否允许被访问
- isLoginRequest 是否登录请求

这个方法提供了自动登录，比如我们获取到token之后实行自动登录

#### FormAuthenticationFilter

基于form表单的账号密码自动登录的过滤器，我们只需要看这个方法就明白，和renren-fast的实现相似：

```java
public class FormAuthenticationFilter extends AuthenticatingFilter {

    public static final String DEFAULT_USERNAME_PARAM = "username";
    public static final String DEFAULT_PASSWORD_PARAM = "password";
    public static final String DEFAULT_REMEMBER_ME_PARAM = "rememberMe";

    protected AuthenticationToken createToken(ServletRequest request, ServletResponse response) {
        String username = getUsername(request);
        String password = getPassword(request);
        return createToken(username, password, request, response);
    }
    /**
     * 公众号：MarkerHub
    **/
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        if (isLoginRequest(request, response)) {
            if (isLoginSubmission(request, response)) {
                if (log.isTraceEnabled()) {
                    log.trace("Login submission detected.  Attempting to execute login.");
                }
                return executeLogin(request, response);
            } else {
                if (log.isTraceEnabled()) {
                    log.trace("Login page view.");
                }
                //allow them to see the login page ;)
                return true;
            }
        } else {
            if (log.isTraceEnabled()) {
                log.trace("Attempting to access a path which requires authentication.  Forwarding to the " +
                        "Authentication url [" + getLoginUrl() + "]");
            }

            saveRequestAndRedirectToLogin(request, response);
            return false;
        }
    }
}
```

onAccessDenied调用executeLogin方法。默认的token是UsernamepasswordToken。

