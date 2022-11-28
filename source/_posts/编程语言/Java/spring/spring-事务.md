---
title: spring-06-事务
date: 2021-05-21 08:07:39
tags:
---

# 事务的应用

## 数据库的事务

> 数据库事务（Transaction，简写为 TX）是数据库管理系统执行过程中的一个逻辑单位，是可以提交或回滚的工作的原子单元。当事务对数据库进行多次更改时，要么在提交事务时所有更改都成功，要么在回滚事务时所有更改都被撤消。

## Mysql 中的事务

- `START TRANSACTION`或 `BEGIN`开始新事务。
- `COMMIT` 提交当前事务。
- `ROLLBACK` 回滚当前事务。
- `SET autocommit` 禁用或启用当前会话的默认自动提交模式。

**默认情况下，Mysql 是自动提交的模式，所有语句会立即提交**

## JDBC 中的事务

**JDBC** 是 Java 语言中用来规范客户端程序如何来访问数据库的应用程序接口，提供了查询和更新数据库中数据的方法。JDBC 也是 Sun Microsystems 的商标（现在属于 Oracle），是面向关系型数据库的。

上面说到，Mysql 是默认自动提交的，所以 JDBC 中事务事务的第一步，需要**禁用自动提交：**

```
con.setAutoCommit(false);
```

**提交事务：**

```
con.commit();
```

**回滚事务：**

```
con.rollback();
```

**一个完整流程的例子（摘自 Oracle JDBC 文档）：**

```java
public void updateCoffeeSales(HashMap<String, Integer> salesForWeek)
    throws SQLException {

    PreparedStatement updateSales = null;
    PreparedStatement updateTotal = null;

    String updateString =
        "update " + dbName + ".COFFEES " +
        "set SALES = ? where COF_NAME = ?";

    String updateStatement =
        "update " + dbName + ".COFFEES " +
        "set TOTAL = TOTAL + ? " +
        "where COF_NAME = ?";

    try {
        con.setAutoCommit(false); //第一步
        updateSales = con.prepareStatement(updateString);
        updateTotal = con.prepareStatement(updateStatement);

        for (Map.Entry<String, Integer> e : salesForWeek.entrySet()) {
            updateSales.setInt(1, e.getValue().intValue());
            updateSales.setString(2, e.getKey());
            updateSales.executeUpdate();
            updateTotal.setInt(1, e.getValue().intValue());
            updateTotal.setString(2, e.getKey());
            updateTotal.executeUpdate();
            con.commit(); //第二步
        }
    } catch (SQLException e ) {
        JDBCTutorialUtilities.printSQLException(e);
        if (con != null) {
            try {
                System.err.print("Transaction is being rolled back");
                con.rollback(); //第三步
            } catch(SQLException excep) {
                JDBCTutorialUtilities.printSQLException(excep);
            }
        }
    } finally {
        if (updateSales != null) {
            updateSales.close();
        }
        if (updateTotal != null) {
            updateTotal.close();
        }
        con.setAutoCommit(true);
    }
}
```

## Spring 事务管理器简介

Spring 为事务管理提供了统一的抽象，有以下优点：<!--注意：Spring只提供抽象，不提供实现-->

- 跨不同事务 API（例如 Java 事务 API（JTA），JDBC，Hibernate，Java 持久性 API（JPA）和 Java 数据对象（JDO））的一致编程模型。
- 支持声明式事务管理（注解形式）
- 与 JTA 之类的复杂事务 API 相比， 用于程序化事务管理的 API 更简单
- 和 Spring 的 Data 层抽象集成方便（比如 Spring - Hibernate/Jdbc/Mybatis/Jpa...）

[Spring 的事务管理器只是一个接口 / 抽象，不同的 DB 层框架（其实不光是 DB 类框架，支持事务模型的理论上都可以使用这套抽象） 可能都需要实现此标准才可以更好的工作]()， 

核心接口是`org.springframework.transaction.support.AbstractPlatformTransactionManager`，其代码位于`spring-tx`模块中，比如 Hibernate 中的实现为：`org.springframework.orm.hibernate4.HibernateTransactionManager`

### 使用方式

事务，自然是控制业务的，在一个业务流程内，往往希望保证原子性，要么全成功要么全失败。

所以事务一般是加载`@Service`层，一个 Service方法内调用了多个数据库操作（比如 Dao），在 Service 结束后事务自动提交，如有异常抛出则事务回滚。

这也是 Spring 事务管理的基本使用原则。

#### 注解

在被 Spring 管理的类头上增加`@Transactional`注解，即可对该类下的所有方法开启事务管理。事务开启后，方法内的操作无需手动开启 / 提交 / 回滚事务，一切交给 Spring 管理即可。

```java
@Service
@Transactional
public class TxTestService{
    
    @Autowired
    private OrderRepo orderRepo;

    public void submit(Order order){
        orderRepo.save(order);
    }
}
```

也可以只在方法上配置，方法配置的优先级是大于类的

```java
@Service
public class TxTestService{
    
    @Autowired
    private OrderRepo orderRepo;


    @Transactional
    public void submit(Order order){
        orderRepo.save(order);
    }
}
```

#### TransactionTemplate

TransactionTemplate 这中方式，其实和使用注解形式的区别不大，其核心功能也是由 TransactionManager 实现的，这里只是换了个入口

```java
public <T> T execute(TransactionCallback<T> action) throws TransactionException {
    if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager) {
        return ((CallbackPreferringPlatformTransactionManager) this.transactionManager).execute(this, action);
    }
    else {
        //获取事务信息
        TransactionStatus status = this.transactionManager.getTransaction(this);
        T result;
        try {
            //执行业务代码
            result = action.doInTransaction(status);
        }
        //处理异常回滚
        catch (RuntimeException ex) {
            // Transactional code threw application exception -> rollback
            rollbackOnException(status, ex);
            throw ex;
        }
        catch (Error err) {
            // Transactional code threw error -> rollback
            rollbackOnException(status, err);
            throw err;
        }
        catch (Exception ex) {
            // Transactional code threw unexpected exception -> rollback
            rollbackOnException(status, ex);
            throw new UndeclaredThrowableException(ex, "TransactionCallback threw undeclared checked exception");
        }
        //提交事务
        this.transactionManager.commit(status);
        return result;
    }
}
```

#### XML 配置 tx:advice

过于古老，不做解释

### 隔离级别 (Isolation Level)

事务隔离级别是数据库最重要的特性之一，他保证了脏读 / 幻读等问题不会发生。作为一个事务管理框架自然也是支持此配置的，在 @Transactional 注解中有一个 isolation 配置，可以很方便的配置各个事务的隔离级别，等同于`connection.setTransactionIsolation()`

```
Isolation {
    DEFAULT(-1),
    READ_UNCOMMITTED(1),
    READ_COMMITTED(2),
    REPEATABLE_READ(4),
    SERIALIZABLE(8);
}
```

### 传播行为

传播行为和数据库功能无关，只是事务管理器为了处理复杂业务而设计的一个机制。

比如现在有这样一个调用场景，`A Service -> B Service -> C Service`，但是希望 A/B 在一个事务内，C 是一个独立的事务，同时 C 如果出错，不影响 AB 所在的事务。

此时，就可以通过传播行为来处理；将 C Service 的事务配置为`@Transactional(propagation = Propagation.REQUIRES_NEW)`即可

Spring 支持以下几种传播行为：

| **PROPAGATION TYPE**          | **DESCRIPTION**                                              |
| :---------------------------- | :----------------------------------------------------------- |
| **PROPAGATION_REQUIRED**      | 支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。 如果正要执行的事务不在另外一个事务里，那么就起一个新的事务; 比如说，`ServiceB.methodB`的事务级别定义为PROPAGATION_`REQUIRED`, 那么由于执行`ServiceA.methodA`的时候， `ServiceA.methodA`已经起了事务，这时调用`ServiceB.methodB`，`ServiceB.methodB`看到自己已经运行在`ServiceA.methodA` 的事务内部，就不再起新的事务。而假如`ServiceA.methodA`运行的时`候ServiceB.method`B发现自己没有在事务中，他就会为自己新建一个事务。 这样，在`ServiceA.methodA`或者在`ServiceB.methodB`内的任何地方出现异常，事务都会被回滚。即使`ServiceB.methodB`的事务已经被提交，但是`ServiceA.methodA`在接下来fail要回滚，`ServiceB.methodB`也要回滚。 |
| **PROPAGATION_SUPPORTS**      | 支持当前事务，如果当前没有事务，就以非事务方式执行。 如果当前在事务中，即以事务的形式运行，如果当前不在一个事务中，那么就以非事务的形式运行。 |
| **PROPAGATION_MANDATORY**     | 支持当前事务，如果当前没有事务，就抛出异常。 必须在一个事务中运行，也就是说，他只能被一个父事务调用。否则，他就要抛出异常。 |
| **PROPAGATION_REQUIRES_NEW**  | 新建事务，如果当前存在事务，把当前事务挂起。 比如我们设计`ServiceA.methodA`的事务级别为PROPAGATION_`REQUIRED`，`ServiceB.methodB`的事务级别为PROPAGATION_`REQUIRES_NEW`，那么当执行到`ServiceB.methodB`的时候，`ServiceA.methodA`所在的事务就会挂起，`ServiceB.methodB`会起一个新的事务，等待`ServiceB.methodB`的事务完成以后，他才继续执行。他与PROPAGATION_`REQUIRED `的事务区别在于事务的回滚程度了。因为`ServiceB.methodB`是新起一个事务，那么就是存在两个不同的事务。如果`ServiceB.methodB`已经提交，那么`ServiceA.methodA`失败回滚，`ServiceB.methodB`是不会回滚的。如果`ServiceB.methodB`失败回滚，如果他抛出的异常被`ServiceA.methodA`捕获，`ServiceA.methodA`事务仍然可能提交。 |
| **PROPAGATION_NOT_SUPPORTED** | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。 不支持当前事务。比如`ServiceA.methodA`的事务级别是PROPAGATION_`REQUIRED`，而`ServiceB.methodB`的事务级别是PROPAGATION_`NOT_SUPPORTED`，那么当执行到`ServiceB.methodB`时，`ServiceA.methodA`的事务挂起，而`ServiceB.methodB`则以非事务的方式运行完，再继续`ServiceA.methodA`的事务。 |
| **PROPAGATION_NEVER**         | 以非事务方式执行，如果当前存在事务，则抛出异常。 不能在事务中运行。假设`ServiceA.methodA`的事务级别是PROPAGATION_`REQUIRED`，而ServiceB.methodB的事务级别是PROPAGATION_`NEVER`， 那么`ServiceB.methodB`就要抛出异常了。 |
| **PROPAGATION_NESTED**        | 支持当前事务，新增Savepoint点，与当前事务同步提交或回滚。 理解Nested的关键是savepoint。他与PROPAGATION_`REQUIRES_NEW`的区别是，PROPAGATION_`REQUIRES_NEW`另起一个事务，将会与他的父事务相互独立，而Nested的事务和他的父事务是相依的，他的提交是要等和他的父事务一块提交的。也就是说，如果父事务最后回滚，他也要回滚的。 而`Nested`事务的好处也是他有一个savepoint。 |



### 回滚策略

@Transactional 中有 4 个配置回滚策略的属性，分为 Rollback 策略，和 NoRollback 策略

**默认情况下，RuntimeException 和 Error 这两种异常会导致事务回滚，普通的 Exception（需要 Catch 的）异常不会回滚。**

#### Rollback

配置需要回滚的异常类

```
# 异常类Class
Class<? extends Throwable>[] rollbackFor() default {};
# 异常类ClassName，可以是FullName/SimpleName
String[] rollbackForClassName() default {};
```

#### NoRollback

针对一些要特殊处理的业务逻辑，比如插一些日志表，或者不重要的业务流程，希望就算出错也不影响事务的提交。

可以通过配置 NoRollbackFor 来实现，让某些异常不影响事务的状态。

```
# 异常类Class
Class<? extends Throwable>[] noRollbackFor() default {};
# 异常类ClassName，可以是FullName/SimpleName
String[] noRollbackForClassName() default {};

复制代码
```

### 只读控制

设置当时事务的只读标示，等同于`connection.setReadOnly()`

### 关键名词解释

| 名词                              | 概念                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| PlatformTransactionManager        | 事务管理器，管理事务的各生命周期方法，简称 TxMgr             |
| TransactionAttribute              | 事务属性, 包含隔离级别，传播行为, 是否只读等信息，简称 TxAttr |
| TransactionStatus                 | 事务状态，包含当前事务、挂起等信息，简称 TxStatus            |
| TransactionInfo                   | 事务信息，内含 TxMgr, TxAttr, TxStatus 等信息，简称 TxInfo   |
| TransactionSynchronization        | 事务同步回调，内含多个钩子方法，简称 TxSync / transaction synchronization |
| TransactionSynchronizationManager | 事务同步管理器，维护当前线程事务资源，信息以及 TxSync 集合   |

### 基本原理

```
public void execute(TxCallback txCallback){
    //获取连接
    Connection connection = acquireConnection();
    try{
        //执行业务代码
        doInService();
        //提交事务
        connection.commit();
    }catch (Exception e){
        //回滚事务
        rollback(connection);
    }finally {
        //释放连接
        releaseConnection(connection);
    }
}
```

Spring 事务管理的基本原理就是以上代码，获取连接 -> 执行代码 -> 提交 / 回滚事务。Spring 只是将这个流程给抽象出来了，所有事务相关的操作都交由 TransactionManager 去实现，然后封装一个**模板形式的入口**来执行

比如`org.springframework.transaction.support.TransactionTemplate`的实现：

```
		@Override
    public <T> T execute(TransactionCallback<T> action) throws TransactionException {
        if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager) {
            return ((CallbackPreferringPlatformTransactionManager) this.transactionManager).execute(this, action);
        }
        else {
            //通过事务管理器获取事务
            TransactionStatus status = this.transactionManager.getTransaction(this);
            T result;
            try {
                //执行业务代码
                result = action.doInTransaction(status);
            }
            //处理异常回滚
            catch (RuntimeException ex) {
                // Transactional code threw application exception -> rollback
                rollbackOnException(status, ex);
                throw ex;
            }
            catch (Error err) {
                // Transactional code threw error -> rollback
                rollbackOnException(status, err);
                throw err;
            }
            catch (Exception ex) {
                // Transactional code threw unexpected exception -> rollback
                rollbackOnException(status, ex);
                throw new UndeclaredThrowableException(ex, "TransactionCallback threw undeclared checked exception");
            }
            //提交事务
            this.transactionManager.commit(status);
            return result;
        }
    }
```

注解形式的事务（@Transactional），实现机制也是一样，基于 Spring 的 AOP，将上面 Template 的模式换成了自动的 AOP，在 AOP 的 Interceptor（`org.springframework.transaction.interceptor.TransactionInterceptor`）中来执行这套流程：

```
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
            throws Throwable {

        // If the transaction attribute is null, the method is non-transactional.
        final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
        //获取事务管理器
        final PlatformTransactionManager tm = determineTransactionManager(txAttr);
        final String joinpointIdentification = methodIdentification(method, targetClass);

        if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
            // Standard transaction demarcation with getTransaction and commit/rollback calls.
            //创建事务
            TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
            Object retVal = null;
            try {
                // This is an around advice: Invoke the next interceptor in the chain.
                // This will normally result in a target object being invoked.
                //执行被“AOP”的代码
                retVal = invocation.proceedWithInvocation();
            }
            catch (Throwable ex) {
                // target invocation exception
                //处理异常回滚
                completeTransactionAfterThrowing(txInfo, ex);
                throw ex;
            }
            finally {
                //清除资源
                cleanupTransactionInfo(txInfo);
            }
            
            //提交事务
            commitTransactionAfterReturning(txInfo);
            return retVal;
        }
   ....
}

复制代码
```

### 复杂流程下的事务传播 / 保持相同事务的关键：

- 对于复杂一些的业务流程，会出现各种类之间的调用，Spring 是如何做到保持同一个事务的？
  -  其实基本原理很简单，只需要将当前事务（Connection）隐式的保存至事务管理器内，后续方法在执行 JDBC 操作前，从事务管理器内获取即可：
  -  比如`HibernateTemplate`中的`SessionFactory`中的`getCurrentSession`，这里的`getCurrentSession`就是从（可能是间接的）Spring 事务管理器中获取的
  -  **Spring 事务管理器将处理事务时的相关临时资源（Connection 等）存在`org.springframework.transaction.support.TransactionSynchronizationManager`中，通过 ThreadLocal 维护**

```java
public abstract class TransactionSynchronizationManager {

    private static final Log logger = LogFactory.getLog(TransactionSynchronizationManager.class);

    private static final ThreadLocal<Map<Object, Object>> resources =
            new NamedThreadLocal<Map<Object, Object>>("Transactional resources");

    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
            new NamedThreadLocal<Set<TransactionSynchronization>>("Transaction synchronizations");

    private static final ThreadLocal<String> currentTransactionName =
            new NamedThreadLocal<String>("Current transaction name");

    private static final ThreadLocal<Boolean> currentTransactionReadOnly =
            new NamedThreadLocal<Boolean>("Current transaction read-only status");

    private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
            new NamedThreadLocal<Integer>("Current transaction isolation level");

    private static final ThreadLocal<Boolean> actualTransactionActive =
            new NamedThreadLocal<Boolean>("Actual transaction active");
    ...
}
```

针对一些复杂场景，嵌套事务 + 独立事务，涉及到挂起（suspend），恢复（resume）的情况，相关资源也是存储在**`TransactionSynchronizationManager`** 中的，方便嵌套事务的处理。

比如 A->B 时，A 方法已经开启了事务，并将当前事务资源绑定在**`TransactionSynchronizationManager`，**那么执行 B 之前，会检测当前是否已经存在事务；检测方式就是从**`TransactionSynchronizationManager`**查找并检测状态，如果已经在事务内，那么就根据不同的传播行为配置来执行不同的逻辑，对于 REQUIRES_NEW 等传播行为的处理会麻烦一些，会涉及到 “挂起（suspend）” 和恢复 (resume) 的操作。

# 事务失效的4种场景

## @Transactional注解的方法为非public

### 案例

TestServiceImpl#insertTestWrongModifier()虽然被@Transactional修饰，但是不是public访问符

```java
@Component
public class TestServiceImpl {
    @Transactional
    void insertTestWrongModifier() {
        int re = testMapper.insert(new Test(10,20,30));
        if (re > 0) {
            throw new NeedToInterceptException("need intercept");
        }
        testMapper.insert(new Test(210,20,30));
    }
}
```

测试用例

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class DemoApplicationTests {
   @Resource
   TestServiceImpl testService;

   @Test
   public void  testInvoke(){
       //调用@Transactional标注的默认访问符方法
        testService.insertTestWrongModifier();
   }
}
```

以上的访问方式，导致事务没开启，因此在方法抛出异常时，testMapper.insert(new Test(10,20,30));操作不会进行回滚。如果`TestServiceImpl#insertTestWrongModifier`方法改为public的话将会正常开启事务，testMapper.insert(new Test(10,20,30));将会进行回滚。

### 原理分析

`@Transactional`是基于动态代理实现的，在bean初始化过程中，对含有`@Transactional`标注的bean实例创建代理对象，标注`@Transactional`的方法如果修饰符不是public，那么将不会对bean进行代理对象创建或者不会对方法进行代理调用。

`@Transactional`注解实现原理中，介绍了如何判定一个bean是否创建代理对象，大概逻辑是：

> 根据spring创建的一个aop切点`BeanFactoryTransactionAttributeSourceAdvisor`实例，遍历当前bean的class的方法对象，判断方法上面的注解信息是否包含`@Transactional`，如果当前bean的任何一个方法包含`@Transactional`注解信息，那么就是适配这个`BeanFactoryTransactionAttributeSourceAdvisor`切点。则当前bean需要创建代理对象，然后代理对象为我们管理事务开闭逻辑。

spring源码中，在拦截bean的创建过程，寻找bean适配的切点时，运用到`AopUtils#canApply(org.springframework.aop.Pointcut, java.lang.Class<?>, boolean)`的方法，目的就是寻找方法上面的@Transactional信息，如果有，就表示切点BeanFactoryTransactionAttributeSourceAdvisor能够应用（canApply）到bean中，

```java
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
   Assert.notNull(pc, "Pointcut must not be null");
   if (!pc.getClassFilter().matches(targetClass)) {
      return false;
   }

   MethodMatcher methodMatcher = pc.getMethodMatcher();
   if (methodMatcher == MethodMatcher.TRUE) {
      // No need to iterate the methods if we're matching any method anyway...
      return true;
   }

   IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
   if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
      introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
   }

    //遍历class的方法对象
   Set<Class<?>> classes = new LinkedHashSet<Class<?>>(ClassUtils.getAllInterfacesForClassAsSet(targetClass));
   classes.add(targetClass);
   for (Class<?> clazz : classes) {
      Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
      for (Method method : methods) {
         if ((introductionAwareMethodMatcher != null &&
               introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions)) ||
             //适配查询方法上的@Transactional注解信息  
             methodMatcher.matches(method, targetClass)) {
            return true;
         }
      }
   }

   return false;
}
```

我们可以在上面的方法打断点，最终上面的代码还会调用如下方法来判断。在下面的方法上断点，回头看看方法调用堆栈也是不错的方式跟踪。

```
AbstractFallbackTransactionAttributeSource#getTransactionAttribute
```

- `AbstractFallbackTransactionAttributeSource#computeTransactionAttribute`

```java
protected TransactionAttribute computeTransactionAttribute(Method method, Class<?> targetClass) {
   // Don't allow no-public methods as required.
   //非public 方法，返回@Transactional信息一律是null
   if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
      return null;
   }
   //后面省略.......
 }
```



- 以一开始的测试代码为例，如果正常的修饰符的testService是经过cglib创建的代理对象


![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufWFWRfYhnMbqZeQOpFticE0Axp7cJAbcFdWiackBtAejpe4iaDLb15vrtU7c62ibuvFNPVo80qLOvczQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 如果class中的方法都是非public的那么将不是代理对象


![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbufWFWRfYhnMbqZeQOpFticE0oDNLPEFoUyIDeAad2EpRbgActMzE1SfRbRPqXEBJINbLLsVAnv6fYA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



### 举一反三

考虑一种情况，如下面代码所示。两个方法都被@Transactional注解标注，但是一个有public修饰符一个没有，那么这种情况一定会创建代理对象，因为至少有一个public修饰符的@Transactional注解标注方法。

创建了代理对象，insertTestWrongModifier就会开启事务吗？答案是不会。

```java
@Component
public class TestServiceImpl implements TestService {
    @Resource
    TestMapper testMapper;

    @Override
    @Transactional
    public void insertTest() {
        int re = testMapper.insert(new Test(10,20,30));
        if (re > 0) {
            throw new NeedToInterceptException("need intercept");
        }
        testMapper.insert(new Test(210,20,30));
    }
    
    @Transactional
    void insertTestWrongModifier() {
        int re = testMapper.insert(new Test(10,20,30));
        if (re > 0) {
            throw new NeedToInterceptException("need intercept");
        }
        testMapper.insert(new Test(210,20,30));
    }
}
```

原因是在动态代理对象进行代理逻辑调用时，在cglib创建的代理对象的拦截函数中`CglibAopProxy.DynamicAdvisedInterceptor#intercept`，有一个逻辑如下，目的是获取当前被代理对象的当前需要执行的method适配的aop逻辑。

```
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
```

而针对@Transactional注解查找aop逻辑过程，相似地，也是执行一次

```
AbstractFallbackTransactionAttributeSource#getTransactionAttribute
```

- `AbstractFallbackTransactionAttributeSource#computeTransactionAttribute`

也就是说还需要找一个方法上的@Transactional注解信息，没有的话就不执行代理@Transactional对应的代理逻辑，直接执行方法。没有了@Transactional注解代理逻辑，就无法开启事务，这也是上一篇已经讲到的。

## 在类内部调用内部@Transactional标注的方法

入口为 main 方法，在 testTx 方法中配置了 @Transactional 注解，同时在插入数据后抛出 RuntimeException 异常，但是方法执行后插入的数据并没有回滚，竟然插入成功了

```java
public xxxService{
    public void main(){
       //内部调用事务方法
        testTx();
    }

    @Transactional
    public void testTx(){
        UrlMappingEntity urlMappingEntity = new UrlMappingEntity();
        urlMappingEntity.setUrl("http://www.baidu.com");

        urlMappingRepository.save(urlMappingEntity);
        if(true){
            throw new RuntimeException();
        }
    }
}
```

这里不生效的原因是因为入口的方法 / 类没有增加 @Transaction 注解，由于 Spring 的事务管理器也是基于 AOP 实现的，不管是 Cglib(ASM) 还是 Jdk 的动态代理，本质上也都是子类机制；在同类之间的方法调用会直接调用本类代码，不会执行动态代理曾的代码；由于入口方法`main`没有增加代理注解，所以`textTx`方法上增加的事务注解并不会生效

解决方法：

1. 方法testTx是一个内部方法，使用手动开始事务，不能用注解

2. 通过在方法内部获得当前类代理对象的方式，通过代理对象调用方法B

   - springboot启动类加上注解:`@EnableAspectJAutoProxy(exposeProxy = true)`

   - 方法test内部获取代理对象调用方法

     ```java
     public void main(){
     		xxxServiceImpl proxyObj = (xxxServiceImpl) AopContext.currentProxy();
         proxyObj.testTx();
     }
     ```

3. xxxService维护一个自己的xxxService，这样就有代理对象了。<!--注意使用延迟加载避免循环依赖-->

   ```
   @Autowire
   private xxxService service;
   
   public void main(){
       //内部调用事务方法
       service.testTx();
   }
   ```
   



### 原理分析

既然事务管理是基于动态代理对象的代理逻辑实现的，那么如果在类内部调用类内部的事务方法，这个调用事务方法的过程并不是通过代理对象来调用的，而是直接通过this对象来调用方法，绕过的代理对象，肯定就是没有代理逻辑了。

## 异步/跨线程后，事务失效

比如在一个事务方法中，开启了子线程操作库，那么此时子线程的事务和主线程事务是不同的。

因为在 Spring 的事务管理器中，事务相关的资源（连接，session，事务状态之类）都是存放在 TransactionSynchronizationManager 中的，通过 ThreadLocal 存放，如果跨线程的话就无法保证一个事务了

```java
# TransactionSynchronizationManager.java
private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");
private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
        new NamedThreadLocal<>("Transaction synchronizations");
private static final ThreadLocal<String> currentTransactionName =
        new NamedThreadLocal<>("Current transaction name");
private static final ThreadLocal<Boolean> currentTransactionReadOnly =
        new NamedThreadLocal<>("Current transaction read-only status");
private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
        new NamedThreadLocal<>("Current transaction isolation level");
private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");
```

## 事务方法内部捕捉了异常，但没有throw Runtime Exception或者Error

```
org.springframework.transaction.UnexpectedRollbackException: 
Transaction silently rolled back because it has been marked as rollback-only
```

这个异常是由于在同一个事务内，多个事务方法之间调用，子方法抛出异常，但又被父方法忽略了导致的。

因为子方法抛出了异常，Spring 事务管理器会将当前事务标为失败状态，准备进行回滚，可是当子方法执行完毕出栈后，父方法又忽略了此异常，待方法执行完毕后正常提交时，事务管理器会检查回滚状态，若有回滚标示则抛出此异常。具体可以参考`org.springframework.transaction.support.AbstractPlatformTransactionManager#processCommit`

### 案例分析

```java
A -> B
# A Service(@Transactional):
public void testTx(){
    urlMappingRepo.deleteById(98l);
    try{
        txSubService.testSubTx();
    }catch (Exception e){
      	//捕获异常，但是没有抛出
        e.printStackTrace();
    }
}

# B Service(@Transactional)
public void testSubTx(){
    if(true){
        throw new RuntimeException();
    }
}
```

### 原理分析

Traditional默认是捕捉Runtime Exception和Error的，如果程序之运行时发生的异常不是Runtime Exception或者Error，是不会被回滚的，所以要rollbackfor exception

# Spring对事务异常的处理

java阿里巴巴规范提示：方法【edit】需要在Transactional注解指定rollbackFor或者在方法中显示的rollback。

## jdk异常分类



## @Transactional 正确的写法

- Spring默认对unchecked exceptions (RuntimeException及其子类和Error) 回滚。

- 如果是 checked Exception，分4种情况：
  - catch了不做任何异常抛出，Spring没有捕获到显然不用回滚；
  - catch了又抛出RuntimeException，Spring捕获到肯定会回滚；
  - catch了但又抛出了unchecked Exception，Spring默认不回滚；
  - 不catch直接抛出unchecked Exception,  Spring默认也是不回滚。



Spring框架的事务基础架构代码将默认地只在抛出Runtime Exception和unchecked exceptions时才标识事务回滚。从事务方法中抛出的Checked exceptions将不被标识进行事务回滚。

1. 让checked例外也回滚：在整个方法前加上 `@Transactional(rollbackFor=Exception.class)`
2. 让unchecked例外不回滚：`@Transactional(notRollbackFor=RunTimeException.class)`
3. 不需要事务管理的(只查询的)方法：`@Transactional(propagation=Propagation.NOT_SUPPORTED)`

注意：如果异常被 `try {} catch {}` 了，事务就不回滚了，如果想让事务回滚必须再往外抛 `try {} catch {throw Exception}` 。

## 注意

1、Spring团队的建议是你在具体的类（或类的方法）上使用 @Transactional 注解，而不要使用在类所要实现的任何接口上。

你当然可以在接口上使用 @Transactional 注解，但是这将只能当你设置了基于接口的代理时它才生效。因为注解是不能继承的，这就意味着如果你正在使用基于类的代理时，那么事务的设置将不能被基于类的代理所识别，而且对象也将不会被事务代理所包装（将被确认为严重的）

2、@Transactional 注解标识的方法，处理过程尽量的简单。

尤其是带锁的事务方法，能不放在事务里面的最好不要放在事务里面。

可以将常规的数据库查询操作放在事务前面进行，而事务内进行增、删、改、加锁查询等操作。