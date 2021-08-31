---
title: shiro应用-02-权限配置
date: 2021-05-07 09:17:57
tags:
---

### 配置位置

认证和授权的存储和交互都在AuthorizingReaml中

```java
public class AuthRealm extends AuthorizingRealm {

    /**
     * @Lazy 延迟注入，不然redis注解会因为注入顺序问题失效
     */
    @Autowired
    @Lazy
    private UserService userService;

    /**
     * 授权
     * @param principals
     * @return
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        User user = (User) principals.fromRealm(this.getClass().getName()).iterator().next();
        List<String> permissionList = new ArrayList<>();
        List<String> roleNameList = new ArrayList<>();
        Set<Role> roleSet = user.getRoles();
        if (CollectionUtils.isNotEmpty(roleSet)) {
            for(Role role : roleSet) {
                roleNameList.add(role.getName());
                Set<Permission> permissionSet = role.getPermissions();
                if (CollectionUtils.isNotEmpty(permissionSet)) {
                    for (Permission permission : permissionSet) {
                        permissionList.add(permission.getName());
                    }
                }
            }
        }
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        info.addStringPermissions(permissionList);
        info.addRoles(roleNameList);
        return info;
    }

    /**
     * 认证登录
     * @param token
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        UsernamePasswordToken usernamePasswordToken = (UsernamePasswordToken) token;
        String username = usernamePasswordToken.getUsername();
        User user = userService.findByUsername(username);
        return new SimpleAuthenticationInfo(user, user.getPassword(), this.getClass().getName());
    }
}
```

### 执行位置

shiro 中的AuthorizingRealm有2个方法doGetAuthorizationInfo()和doGetAuthenticationInfo(),一般实际开发中继承AuthorizingRealm类然后重写doGetAuthorizationInfo和doGetAuthenticationInfo。     

- doGetAuthenticationInfo方法是在用户登录的时候调用的也就是执SecurityUtils.getSubject().login（）的时候调用，即登录验证
- doGetAuthorizationInfo方法是调用SecurityUtils.getSubject().isPermitted（）这个方法时会调用doGetAuthorizationInfo（），
  - @RequiresPermissions这个注解起始就是在执行SecurityUtils.getSubject().isPermitted（）。
  - 某个方法上加上@RequiresPermissions，那么访问这个方法就会自动调用SecurityUtils.getSubject().isPermitted（），从而区调用doGetAuthorizationInfo

#### 踩的坑

调用链：isPermitted -> getAuhtorizationInfo -> doGetAuthorizationInfo

但是，如果不做单点登录，一个账号被多个用户登录，那么账户信息会被cache保存，在getAuhtorizationInfo会判断，如果账户信息被cache了，就不执行doGetAuthorizationInfo