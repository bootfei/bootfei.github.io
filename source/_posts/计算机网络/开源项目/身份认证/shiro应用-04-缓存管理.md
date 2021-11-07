---
title: shiro应用-04-缓存管理
date: 2021-05-19 19:13:10
tags:
---

在每一次请求需要权限的时候总是会调用授权的方法查询数据库，这样的话性能很低，因此我们可以使用缓存管理器，来达到这种要求，在Shiro中有一个内存缓存管理器，内部就是使用Map实现的，但是这种缓存并不能实现跨JVM（分布式），因此我们可以使用Redis自定义一个缓存管理器，步骤如下：

## Redis缓存

### 实现RedisCache，用于实现对授权信息的缓存

```java
/**
 * Redis的Cache
 */
public class RedisCache<K,V> implements Cache<K,V> {

     private RedisTemplate redisTemplate;

     /**
     * 存储在redis中的hash中的key
     */
     private String name;

     private final static String COMMON_NAME="shiro-demo";


     public RedisCache(RedisTemplate redisTemplate, String name) {
         this.redisTemplate = redisTemplate;
         this.name=COMMON_NAME+":"+name;
     }

     /**
     * 获取指定的key的缓存
     * @param k
     * @return
     * @throws CacheException
     */
     @Override
     public V get(K k) throws CacheException {
     		return (V) redisTemplate.opsForHash().get(name,k);
     }

     /**
     * 添加缓存
     * @param k
     * @param v
     * @return
     * @throws CacheException
     */
     @Override
     public V put(K k, V v) throws CacheException {
         redisTemplate.opsForHash().put(name, k, v);
         //设置过期时间
         return v;
     }

     /**
     * 删除指定key的缓存
     * @param k 默认是principle对象，在AuthorizingRealm中设置
     */
     @Override
     public V remove(K k) throws CacheException {
         V v = this.get(k);
         redisTemplate.opsForHash().delete(name, k);
         return v;
      }

     /**
     * 删除所有的缓存
     */
     @Override
     public void clear() throws CacheException {
     		redisTemplate.delete(name);
      }

     /**
     * 获取总数
     * @return
     */
     @Override
     public int size() {
     		return redisTemplate.opsForHash().size(name).intValue();
      }

     @Override
     public Set<K> keys() {
     		return redisTemplate.opsForHash().keys(name);
     }

     @Override
     public Collection<V> values() {
     		return redisTemplate.opsForHash().values(name);
      }
}
```

### 实现RedisManager

```java

/**
 * Redis的CacheManager
 */
public class RedisCacheManager implements CacheManager {

     @Autowired
     private RedisTemplate redisTemplate;

     @Override
     public <K, V> Cache<K, V> getCache(String s) throws CacheException {
     		return new RedisCache<K,V>(redisTemplate, s);
     }
}
```

### 在配置类中配置上缓存管理器，需要设置到SecurityManager中才能生效

```java
/**
 * 配置缓存管理器，使用自定义的Redis缓存管理器
 */
 @Bean
 public CacheManager cacheManager(){
 		return new RedisCacheManager();
 }

 /**
 * 配置安全管理器
 */
 @Bean
 public SecurityManager securityManager(){
     DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager(userRealm());
     //设置缓存管理器
     securityManager.setCacheManager(cacheManager());
     return securityManager;
 }

```





## **清除缓存**

- 在CachingRelam中有一个清除缓存的方法`org.apache.shiro.realm.CachingRealm#clearCache`，在我们自定义的Realm中覆盖该方法即可，这样就能在**退出或者在业务逻辑中用户的权限改变**的时候能够清除缓存的数据，如下：

```java
    /**
     * 清除CacheManager中的缓存，可以在用户权限改变的时候调用，这样再次需要权限的时候就会重新查询数据库不走缓存了
     */
    public void clearCache() {
        Subject subject = SecurityUtils.getSubject();
        //调用父类的清除缓存的方法
        super.clearCache(subject.getPrincipals());
    }
```



- **除了重写或者覆盖CachingRelam中的方法，根据源码可以知道，真正起作用的方法是`AuthorizingRealm`中的方法`clearCachedAuthorizationInfo`，因此我们也可以重写或者覆盖这个方法，这里不再演示。**





## **实现原理**

- 在Shiro中一旦有地方调用`Subject.hasRole`等校验权限的地方，那么就会检测授权信息，在`org.apache.shiro.realm.AuthorizingRealm#getAuthorizationInfo`的方法中会先缓存中查询是否存在，否则调用授权的方法从数据库中查询，查询之后放入缓存中，源码如下：

```java
 protected AuthorizationInfo getAuthorizationInfo(PrincipalCollection principals) {

        if (principals == null) {
            return null;
        }

        AuthorizationInfo info = null;

        if (log.isTraceEnabled()) {
            log.trace("Retrieving AuthorizationInfo for principals [" + principals + "]");
        }
    
        //获取可用的缓存管理器
        Cache<Object, AuthorizationInfo> cache = getAvailableAuthorizationCache();
        if (cache != null) {
            if (log.isTraceEnabled()) {
                log.trace("Attempting to retrieve the AuthorizationInfo from cache.");
            }
            //获取缓存的key，这里获取的就是principal主体信息
            Object key = getAuthorizationCacheKey(principals);
            //从缓存中获取数据
            info = cache.get(key);
            if (log.isTraceEnabled()) {
                if (info == null) {
                    log.trace("No AuthorizationInfo found in cache for principals [" + principals + "]");
                } else {
                    log.trace("AuthorizationInfo found in cache for principals [" + principals + "]");
                }
            }
        }

        //如果缓存中没有查到
        if (info == null) {
            //调用重写的授权方法，从数据库中查询
            info = doGetAuthorizationInfo(principals);
        //如果查询到了，添加到缓存中
            if (info != null && cache != null) {
                if (log.isTraceEnabled()) {
                    log.trace("Caching authorization info for principals: [" + principals + "].");
                }
                //获取缓存的key
                Object key = getAuthorizationCacheKey(principals);
                //放入缓存
                cache.put(key, info);
            }
        }

        return info;
    }

```



