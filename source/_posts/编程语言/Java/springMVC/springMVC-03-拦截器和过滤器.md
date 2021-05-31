---
title: springMVC-03-拦截器和过滤器
date: 2021-05-31 08:55:22
tags:
---



## 拦截器

### HandlerInterceptor实现登录拦截的原理

SpringBoot通过实现HandlerInterceptor接口实现拦截器，通过实现WebMvcConfigurer接口实现一个配置类，在配置类中注入拦截器，最后再通过@Configuration注解注入配置.

#### 实现HandlerInterceptor接口

实现HandlerInterceptor接口需要实现3个方法：`preHandle`、`postHandle`、`afterCompletion`.

3个方法各自的功能如下：

```java
import blog.entity.User;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

public class UserLoginInterceptor implements HandlerInterceptor {

    /***
     * 在请求处理之前进行调用(Controller方法调用之前)
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("执行了拦截器的preHandle方法");
        try {
            HttpSession session = request.getSession();
            //统一拦截（查询当前session是否存在user）(这里user会在每次登录成功后，写入session)
            User user = (User) session.getAttribute("user");
            if (user != null) {
                return true;
            }
            response.sendRedirect(request.getContextPath() + "login");
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
        //如果设置为false时，被请求时，拦截器执行到此处将不会继续操作
        //如果设置为true时，拦截器将会继续执行后面的操作
    }

    /***
     * 请求处理之后进行调用，但是在视图被渲染之前（Controller方法调用之后）
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("执行了拦截器的postHandle方法");
    }

    /***
     * 整个请求结束之后被调用，也就是在DispatchServlet渲染了对应的视图之后执行（主要用于进行资源清理工作）
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("执行了拦截器的afterCompletion方法");
    }
}
```

preHandle在Controller之前执行，因此拦截器的功能主要就是在这个部分实现：

- 检查session中是否有user对象存在；
- 如果存在，就返回true，那么Controller就会继续后面的操作；
- 如果不存在，就会重定向到登录界面。就是通过这个拦截器，使得Controller在执行之前，都执行一遍preHandle.

#### 实现WebMvcConfigurer接口，注册拦截器

实现WebMvcConfigurer接口来实现一个配置类，将上面实现的拦截器的一个对象注册到这个配置类中.

```java
import blog.interceptor.UserLoginInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class LoginConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //注册TestInterceptor拦截器
        InterceptorRegistration registration = registry.addInterceptor(new UserLoginInterceptor());
        registration.addPathPatterns("/**"); //所有路径都被拦截
        registration.excludePathPatterns(    //添加不拦截路径
                "/login",                    //登录路径
                "/**/*.html",                //html静态资源
                "/**/*.js",                  //js静态资源
                "/**/*.css"                  //css静态资源
        );
    }
}
```

将拦截器注册到了拦截器列表中，并且指明了拦截哪些访问路径，不拦截哪些访问路径，不拦截哪些资源文件；最后再以@Configuration注解将配置注入。

#### 保持登录状态

只需一次登录，如果登录过，下一次再访问的时候就无需再次进行登录拦截，可以直接访问网站里面的内容了。

在正确登录之后，就将user保存到session中，再次访问页面的时候，登录拦截器就可以找到这个user对象，就不需要再次拦截到登录界面了.

```java
@RequestMapping(value = {"/login"}, method = RequestMethod.GET)
public String loginIndex() {
    return "users/login";
}

@RequestMapping(value = {"/login"}, method = RequestMethod.POST)
public String login(@RequestParam(name = "username")String username, @RequestParam(name = "password")String password, Model model, HttpServletRequest request) {
    User user = userService.getPwdByUsername(username);
    String pwd = user.getPassword();
    if (pwd.equals(password2)) {
        model.addAttribute("user", user);
        request.getSession().setAttribute("user", user);
        return "redirect:/index";
    } else {
        return "users/failed";
    }
}
```

