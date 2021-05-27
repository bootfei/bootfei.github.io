---
title: SpringBoot-0-接口和注解
date: 2021-05-27 15:19:02
tags:
---



<!--springboot注解和接口属于spring，但是springboot大量使用这些注解和接口，所以放在springboot讲解-->



## 注解

### @Conditional

#### 使用

@Conditional是由Spring 4提供的一个新特性，用于根据特定条件来控制Bean的创建行为。而在我们开发基于Spring的应用的时候，难免会需要根据条件来注册Bean。

在业务复杂的情况下，显然需要使用到@Conditional注解来提供更加灵活的条件判断，例如以下几个判断条件：

- 在类路径中是否存在这样的一个类。
- 在Spring容器中是否已经注册了某种类型的Bean（如未注册，我们可以让其自动注册到容器中，上一条同理）。
- 一个文件是否在特定的位置上。
- 一个特定的系统属性是否存在。
- 在Spring的配置文件中是否设置了某个特定的值。

@Conditional(value)中value值，是个Condition类，需要实现match()方法

#### 原理

在Spring Boot中到处都有类似的注解，像@ConditionalOnBean（容器中是否有指定的Bean），@ConditionalOnWebApplication（当前工程是否为一个Web工程）等等，它们都只是@Conditional注解的扩展。

#### 实践

##### 根据配置文件属性注册对应Bean

比如，需要根据命令行传入的系统参数来注册对应的UserDao，就像`java -jar app.jar -DdbType=MySQL`会注册JdbcUserDao，而`java -jar app.jar -DdbType=MongoDB`则会注册MongoUserDao。

```java
public class MySQLDatabaseTypeCondition implements Condition {
	@Override
	public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) {
 		String enabledDBType = System.getProperty("dbType"); // 获得系统参数 dbType
 		// 如果该值等于MySql，则条件成立
 		return (enabledDBType != null && enabledDBType.equalsIgnoreCase("MySql"));
 	}
}


public class MongoDBDatabaseTypeCondition implements Condition {
	@Override
	public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) {
 		String enabledDBType = System.getProperty("dbType");
 		return (enabledDBType != null && enabledDBType.equalsIgnoreCase("MongoDB"));
 	}
}
```


```java
// 根据条件来注册不同的Bean
@Configuration
public class AppConfig {
	@Bean
	@Conditional(MySQLDatabaseTypeCondition.class)
	public UserDAO jdbcUserDAO() {
		return new JdbcUserDAO();
	}
	
	@Bean
	@Conditional(MongoDBDatabaseTypeCondition.class)
	public UserDAO mongoUserDAO() {
		return new MongoUserDAO();
	}
}
```



比如，根据配置文件中的某项属性来决定是否注册MongoDAO，例如`app.dbType`是否等于`MongoDB`

```java
public class MongoDbTypePropertyCondition implements Condition {
	@Override
	public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) {
		String dbType = conditionContext.getEnvironment().getProperty("app.dbType");
		return "MONGO".equalsIgnoreCase(dbType);
	}
}
```



##### 根据是否存在其他类决定是否注册Bean

根据当前工程的类路径中是否存在MongoDB的驱动类来确认是否注册MongoUserDAO。

```java
public class MongoDriverPresentsCondition implements Condition {
	@Override
	public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) {
		try {
			Class.forName("com.mongodb.Server");
			return true;
		} catch (ClassNotFoundException e) {
			return false;
		}
	}
}

public class MongoDriverNotPresentsCondition implements Condition {
	@Override
	public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) {
		try {
			Class.forName("com.mongodb.Server");
			return false;
		} catch (ClassNotFoundException e) {
			return true;
		}
	}
}
```

假如，你想要在UserDAO没有被注册的情况下去注册一个UserDAOBean，那么我们可以定义一个条件类来检查某个类是否在容器中已被注册。

```java
public class UserDAOBeanNotPresentsCondition implements Condition {
	@Override
	public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) {
		UserDAO userDAO = conditionContext.getBeanFactory().getBean(UserDAO.class);
		return (userDAO == null);
	}
}
```

##### 使用组合注解完成自定义注解

以注解的方式来完成条件判断 ，@Profile一样。首先，我们需要定义一个注解类。

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(DatabaseTypeCondition.class)
public @interface DatabaseType {
	String value();
}
```

具体的条件判断逻辑在DatabaseTypeCondition类中，它会根据系统参数`dbType`来判断注册哪一个Bean。

```java
public class DatabaseTypeCondition implements Condition {
	@Override
	public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) {
		Map<String, Object> attributes = metadata
											.getAnnotationAttributes(DatabaseType.class.getName());
		String type = (String) attributes.get("value");
		// 默认值为MySql
		String enabledDBType = System.getProperty("dbType", "MySql");
		return (enabledDBType != null && type != null && enabledDBType.equalsIgnoreCase(type));
	}
}
```

最后，在配置类应用该注解即可。

```java
@Configuration
@ComponentScan
public class AppConfig {
	@Bean
	@DatabaseType("MySql")
	public UserDAO jdbcUserDAO() {
		return new JdbcUserDAO();
	}

	@Bean
	@DatabaseType("mongoDB")
	public UserDAO mongoUserDAO() {
		return new MongoUserDAO();
	}
}
```





##### 自动配置类中的条件注解

------

在spring.factories文件中随便找一个自动配置类，来看看是怎样实现的。MongoDataAutoConfiguration的源码，发现它声明了@ConditionalOnClass注解，通过看该注解的源码后可以发现，这是一个组合了@Conditional的组合注解，它的条件类是OnClassCondition。

```
@Configuration
@ConditionalOnClass({Mongo.class, MongoTemplate.class})
@EnableConfigurationProperties({MongoProperties.class})
@AutoConfigureAfter({MongoAutoConfiguration.class})
public class MongoDataAutoConfiguration {
	....
}

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional({OnClassCondition.class})
public @interface ConditionalOnClass {
    Class<?>[] value() default {};

    String[] name() default {};
}
```

然后，我们开始看OnClassCondition的源码，发现它并没有直接实现Condition接口，只好往上找，发现它的父类SpringBootCondition实现了Condition接口。

```java
class OnClassCondition extends SpringBootCondition implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware {
	.....
}

public abstract class SpringBootCondition implements Condition {
    private final Log logger = LogFactory.getLog(this.getClass());

    public SpringBootCondition() {
    }

    public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String classOrMethodName = getClassOrMethodName(metadata);

        try {
            ConditionOutcome ex = this.getMatchOutcome(context, metadata);
            this.logOutcome(classOrMethodName, ex);
            this.recordEvaluation(context, classOrMethodName, ex);
            return ex.isMatch();
        } catch (NoClassDefFoundError var5) {
          
        } catch (RuntimeException var6) {
          
        }
    }

    public abstract ConditionOutcome getMatchOutcome(ConditionContext var1, AnnotatedTypeMetadata var2);
}
```

SpringBootCondition实现的matches方法依赖于一个抽象方法this.getMatchOutcome(context, metadata)，我们在它的子类OnClassCondition中可以找到这个方法的具体实现。

```java
public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
    ClassLoader classLoader = context.getClassLoader();
    ConditionMessage matchMessage = ConditionMessage.empty();
    // 找出所有ConditionalOnClass注解的属性
    List onClasses = this.getCandidates(metadata, ConditionalOnClass.class);
    List onMissingClasses;
    if(onClasses != null) {
        // 找出不在类路径中的类
        onMissingClasses = this.getMatches(onClasses, OnClassCondition.MatchType.MISSING, classLoader);
        // 如果存在不在类路径中的类，匹配失败
        if(!onMissingClasses.isEmpty()) {
            return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnClass.class, new Object[0]).didNotFind("required class", "required classes").items(Style.QUOTE, onMissingClasses));
        }

        matchMessage = matchMessage.andCondition(ConditionalOnClass.class, new Object[0]).found("required class", "required classes").items(Style.QUOTE, this.getMatches(onClasses, OnClassCondition.MatchType.PRESENT, classLoader));
    }

    // 接着找出所有ConditionalOnMissingClass注解的属性
    // 它与ConditionalOnClass注解的含义正好相反，所以以下逻辑也与上面相反
    onMissingClasses = this.getCandidates(metadata, ConditionalOnMissingClass.class);
    if(onMissingClasses != null) {
        List present = this.getMatches(onMissingClasses, OnClassCondition.MatchType.PRESENT, classLoader);
        if(!present.isEmpty()) {
            return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnMissingClass.class, new Object[0]).found("unwanted class", "unwanted classes").items(Style.QUOTE, present));
        }

        matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class, new Object[0]).didNotFind("unwanted class", "unwanted classes").items(Style.QUOTE, this.getMatches(onMissingClasses, OnClassCondition.MatchType.MISSING, classLoader));
    }

    return ConditionOutcome.match(matchMessage);
}

// 获得所有annotationType注解的属性
private List<String> getCandidates(AnnotatedTypeMetadata metadata, Class<?> annotationType) {
    MultiValueMap attributes = metadata.getAllAnnotationAttributes(annotationType.getName(), true);
    ArrayList candidates = new ArrayList();
    if(attributes == null) {
        return Collections.emptyList();
    } else {
        this.addAll(candidates, (List)attributes.get("value"));
        this.addAll(candidates, (List)attributes.get("name"));
        return candidates;
    }
}

private void addAll(List<String> list, List<Object> itemsToAdd) {
    if(itemsToAdd != null) {
        Iterator var3 = itemsToAdd.iterator();

        while(var3.hasNext()) {
            Object item = var3.next();
            Collections.addAll(list, (String[])((String[])item));
        }
    }

}    

// 根据matchType.matches方法来进行匹配
private List<String> getMatches(Collection<String> candidates, OnClassCondition.MatchType matchType, ClassLoader classLoader) {
    ArrayList matches = new ArrayList(candidates.size());
    Iterator var5 = candidates.iterator();

    while(var5.hasNext()) {
        String candidate = (String)var5.next();
        if(matchType.matches(candidate, classLoader)) {
            matches.add(candidate);
        }
    }

    return matches;
}
```

关于match的具体实现在MatchType中，它是一个枚举类，提供了PRESENT和MISSING两种实现，前者返回类路径中是否存在该类，后者相反。

```
private static enum MatchType {
    PRESENT {
        public boolean matches(String className, ClassLoader classLoader) {
            return OnClassCondition.MatchType.isPresent(className, classLoader);
        }
    },
    MISSING {
        public boolean matches(String className, ClassLoader classLoader) {
            return !OnClassCondition.MatchType.isPresent(className, classLoader);
        }
    };

    private MatchType() {
    }

    // 跟我们之前看过的案例一样，都利用了类加载功能来进行判断
    private static boolean isPresent(String className, ClassLoader classLoader) {
        if(classLoader == null) {
            classLoader = ClassUtils.getDefaultClassLoader();
        }

        try {
            forName(className, classLoader);
            return true;
        } catch (Throwable var3) {
            return false;
        }
    }

    private static Class<?> forName(String className, ClassLoader classLoader) throws ClassNotFoundException {
        return classLoader != null?classLoader.loadClass(className):Class.forName(className);
    }

    public abstract boolean matches(String var1, ClassLoader var2);
}
```

现在终于真相大白，@ConditionalOnClass的含义是指定的类必须存在于类路径下，MongoDataAutoConfiguration类中声明了类路径下必须含有Mongo.class, MongoTemplate.class这两个类，否则该自动配置类不会被加载。

### @Profile

#### 使用

你想要根据不同的运行环境，来让Spring注册对应环境的数据源Bean，对于这种简单的情况，完全可以使用@Profile注解实现，就像下面代码所示：

```
@Configuration
public class AppConfig {
	@Bean
	@Profile("DEV")
	public DataSource devDataSource() {
		...
	}
	
	@Bean
	@Profile("PROD")
	public DataSource prodDataSource() {
		...
	}
}
```

剩下只需要设置对应的Profile属性即可，设置方法有如下三种：

- 通过`context.getEnvironment().setActiveProfiles("PROD")`来设置Profile属性。
- 通过设定jvm的`spring.profiles.active`参数来设置环境（Spring Boot中可以直接在`application.properties`配置文件中设置该属性）。
- 通过在DispatcherServlet的初始参数中设置。

```xml
<servlet>
	<servlet-name>dispatcher</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	<init-param>
		<param-name>spring.profiles.active</param-name>
		<param-value>PROD</param-value>
	</init-param>
</servlet>
```

#### 原理

通过源码我们可以发现@Profile自身也使用了@Conditional注解。

```java
package org.springframework.context.annotation;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional({ProfileCondition.class}) // 组合了Conditional注解
public @interface Profile {
    String[] value();
}

```



```java
package org.springframework.context.annotation;

class ProfileCondition implements Condition {
    ProfileCondition() {
    }

    // 通过提取出@Profile注解中的value值来与profiles配置信息进行匹配
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        if(context.getEnvironment() != null) {
            MultiValueMap attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
            if(attrs != null) {
                Iterator var4 = ((List)attrs.get("value")).iterator();

                Object value;
                do {
                    if(!var4.hasNext()) {
                        return false;
                    }

                    value = var4.next();
                } while(!context.getEnvironment().acceptsProfiles((String[])((String[])value)));

                return true;
            }
        }

        return true;
    }
}
```





### @EnableAutoConfiguration

@SpringBootApplication中的重要注解，用于发现Spring.factories文件中需要自动装配的类

#### 原理

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({EnableAutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```

##### @Import

@Import（Spring 提供的一个注解，可以导入配置类或者Bean到当前类中）导入了EnableAutoConfigurationImportSelector类，根据名字来看，它应该就是我们要找到的目标了。不过查看它的源码发现它已经被Deprecated了，而官方API中告知我们去查看它的父类AutoConfigurationImportSelector。

```
/** @deprecated */
@Deprecated
public class EnableAutoConfigurationImportSelector extends AutoConfigurationImportSelector {
    public EnableAutoConfigurationImportSelector() {
    }

    protected boolean isEnabled(AnnotationMetadata metadata) {
        return this.getClass().equals(EnableAutoConfigurationImportSelector.class)?((Boolean)this.getEnvironment().getProperty("spring.boot.enableautoconfiguration", Boolean.class, Boolean.valueOf(true))).booleanValue():true;
    }
}
```

##### AutoConfigurationImportSelector#selectImports

由于AutoConfigurationImportSelector的源码太长了，这里我只截出关键的地方，显然方法selectImports是选择自动配置的主入口，它调用了其他的几个方法来加载元数据等信息，最后返回一个包含许多自动配置类信息的字符串数组。

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if(!this.isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    } else {
        try {
            AutoConfigurationMetadata ex = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
            List configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
            configurations = this.removeDuplicates(configurations);
            configurations = this.sort(configurations, ex);
            Set exclusions = this.getExclusions(annotationMetadata, attributes);
            this.checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            configurations = this.filter(configurations, ex);
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            return (String[])configurations.toArray(new String[configurations.size()]);
        } catch (IOException var6) {
            throw new IllegalStateException(var6);
        }
    }
}
```

##### AutoConfigurationImportSelector#getCandidateConfigurations

重点在于方法getCandidateConfigurations()返回了自动配置类的信息列表，而它通过调用SpringFactoriesLoader.loadFactoryNames()来扫描加载含有META-INF/spring.factories文件的jar包，该文件记录了具有哪些自动配置类。

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List configurations = SpringFactoriesLoader
    									.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes 
    found in META-INF spring.factories. 
    If you are using a custom packaging, make sure that file is correct.");
    return configurations;
}
	
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();

    try {
        Enumeration ex = classLoader != null?classLoader.getResources("META-INF/spring.factories"):ClassLoader.getSystemResources("META-INF/spring.factories");
        ArrayList result = new ArrayList();

        while(ex.hasMoreElements()) {
            URL url = (URL)ex.nextElement();
            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
            String factoryClassNames = properties.getProperty(factoryClassName);
            result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
        }

        return result;
    } catch (IOException var8) {
        throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + "META-INF/spring.factories" + "]", var8);
    }
}
```

[![spring.factories](http://wx2.sinaimg.cn/large/63503acbly1fn9iobng7zj21270o9gpg.jpg)](http://wx2.sinaimg.cn/large/63503acbly1fn9iobng7zj21270o9gpg.jpg)











## 接口

@