---
title: springboot-定时任务框架
date: 2021-04-12 19:22:20
tags:
---

# Quartz

Quartz是一个定时任务框架，其他介绍网上也很详尽。这里要介绍一下Quartz里的几个非常核心的接口。

## Scheduler接口

Quartz通过调度器来注册、暂停、删除Trigger和JobDetail。Scheduler还拥有一个SchedulerContext，通过SchedulerContext我们可以获取到触发器和任务的一些信息。

## Trigger接口

Trigger通过cron表达式或是SimpleScheduleBuilder等类，指定任务执行的周期。系统时间走到触发器指定的时间的时候，触发器就会触发任务的执行。

## JobDetail接口

Job接口是真正需要执行的任务。JobDetail接口相当于将Job接口包装了一下，Trigger和Scheduler实际用到的都是JobDetail。

### Job接口

Job接口是真正需要执行的任务，由客户自行定义

# SpringBoot官方文档解读

SpringBoot官方写了`spring-boot-starter-quartz`。使用过SpringBoot的同学都知道这是一个官方提供的启动器，有了这个启动器，集成的操作就会被大大简化。

现在我们来看一看SpingBoot2.2.6官方文档，其中第4.20小节`Quartz Scheduler`就谈到了Quartz，

```
Spring Boot offers several conveniences for working with the Quartz scheduler, including the
spring-boot-starter-quartz “Starter”. If Quartz is available, a Scheduler is auto-configured (through the SchedulerFactoryBean abstraction).
Beans of the following types are automatically picked up and associated with the Scheduler:
• JobDetail: defines a particular Job. JobDetail instances can be built with the JobBuilder API.
• Calendar.
• Trigger: defines when a particular job is triggered.
```

翻译一下：

```
Job可以定义setter(也就是set方法)来注入配置信息。也可以用同样的方法注入普通的bean。
```

下面是文档里给的示例代码，我直接完全照着写，拿到的却是null。不知道是不是我的使用方式有误。后来仔细一想，文档的意思应该是在创建Job对象之后，调用set方法将依赖注入进去。但后面我们是通过框架反射生成的Job对象，这样做反而会搞得更加复杂。最后还是决定采用给Job类加@Component注解的方法。

文档的其他篇幅就介绍了一些配置，但是介绍得也不全面，看了帮助也并不是很大。详细的配置可以参考w3school的[Quartz配置](https://www.w3cschool.cn/quartz_doc/quartz_doc-ml8e2d9m.html)。

# SpringBoot集成Quartz

## 建表

我选择将定时任务的信息保存在数据库中，优点是显而易见的，定时任务不会因为系统的崩溃而丢失。

建表的sql语句在Quartz的github中可以找到，里面有针对每一种常用数据库的sql语句，具体地址是：[Quartz数据库建表sql](https://github.com/quartz-scheduler/quartz/tree/9f9e400733f51f7cb658e3319fc2c140ab8af938/quartz-core/src/main/resources/org/quartz/impl/jdbcjobstore)。

![quartz表](https://segmentfault.com/img/bVbGMZo)

建表以后，可以看到数据库里多了11张表。我们完全不需要关心每张表的具体作用，在添加删除任务、触发器等的时候，Quartz框架会操作这些表。

## 引入依赖

在`pom.xml`里添加依赖。

```
<!-- quartz 定时任务 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
    <version>2.2.6.RELEASE</version>
</dependency>
```

## 配置quartz

在`application.yml`中配置quartz。相关配置的作用已经写在注解上。

```
spring:
  quartz:
      job-store-type: jdbc # 将任务等保存化到数据库
      wait-for-jobs-to-complete-on-shutdown: true # 程序结束时会等待quartz相关的内容结束
      # QuartzScheduler启动时更新己存在的Job,这样就不用每次修改targetObject后删除qrtz_job_details表对应记录
      overwrite-existing-jobs: true
      # 这里居然是个map，搞得智能提示都没有，佛了
      properties:
        org:
          quartz:
            # scheduler相关
            scheduler:        
              instanceName: scheduler # scheduler的实例名
              instanceId: AUTO
            # 持久化相关
            jobStore:
              class: org.quartz.impl.jdbcjobstore.JobStoreTX
              driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
              # 表示数据库中相关表是QRTZ_开头的
              tablePrefix: QRTZ_
              useProperties: false
            # 线程池相关
            threadPool:
              class: org.quartz.simpl.SimpleThreadPool
              threadCount: 10  # 线程数
              threadPriority: 5 # 线程优先级
              threadsInheritContextClassLoaderOfInitializingThread: true
```

## 注册周期性的定时任务

第1节中提到的第一个子需求是在每天0点执行的，是一个周期性的任务，任务内容也是确定的，所以直接在代码里注册JobDetail和Trigger的bean就可以了。当然，这些JobDetail和Trigger也是会被持久化到数据库里。

JobDetail和Trigger

```java
/**
 * Quartz的相关配置，注册JobDetail和Trigger
 * 注意JobDetail和Trigger是org.quartz包下的，不是spring包下的，不要导入错误
 */
@Configuration
public class QuartzConfig {

    @Bean
    public JobDetail jobDetail() {
        JobDetail jobDetail = JobBuilder.newJob(StartOfDayJob.class)
                .withIdentity("start_of_day", "start_of_day")
                .storeDurably()
                .build();
        return jobDetail;
    }

    @Bean
    public Trigger trigger() {
        Trigger trigger = TriggerBuilder.newTrigger()
                .forJob(jobDetail())
                .withIdentity("start_of_day", "start_of_day")
                .startNow()
                // 每天0点执行
                .withSchedule(CronScheduleBuilder.cronSchedule("0 0 0 * * ?"))
                .build();
        return trigger;
    }
}
```

builder类创建了一个JobDetail和一个Trigger并注册成为Spring bean。从摘录的官方文档中，我们已经知道这些bean会自动关联到调度器上。需要注意的是JobDetail和Trigger需要设置组名和自己的名字，用来作为唯一标识。当然，JobDetail和Trigger的唯一标识可以相同，因为他们是不同的类。

Trigger通过cron表达式指定了任务执行的周期。

### QuartzJobBean（Job类逻辑）

JobDetail里有一个StartOfDayJob类，这个类就是Job接口的一个实现类，里面定义了任务的具体内容

```java
@Component
public class StartOfDayJob extends QuartzJobBean {
    private StudentService studentService;

    @Autowired
    public StartOfDayJob(StudentService studentService) {
        this.studentService = studentService;
    }

    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext)
            throws JobExecutionException {
        // 任务的具体逻辑
    }
}
```

> 这里面有一个小问题，上面用builder创建JobDetail时，传入了StartOfDayJob.class，按常理推测，应该是Quartz框架通过反射创建StartOfDayJob对象，再调用executeInternal()执行任务。这样依赖，这个Job是Quartz通过反射创建的，即使加了注解@Component，这个StartOfDayJob对象也不会被注册到ioc容器中，更不可能实现依赖的自动装配。
>
> 网上很多博客也是这么介绍的。但是根据我的实际测试，这样写可以完成依赖注入，但我还不知道它的实现原理。

![编写定时任务](https://segmentfault.com/img/bVbGMZp)

![依赖注入成功](https://segmentfault.com/img/bVbGMZq)

## 注册无周期性的定时任务

第1节中提到的第二个子需求是学生请假，显然请假是不定时的，一次性的，而且不具有周期性。

4.5节与4.4节大体相同，但是有两点区别：

- Job类需要获取到一些数据用于任务的执行；
- 任务执行完成后删除Job和Trigger。

业务逻辑是在老师批准学生的请假申请时，向调度器添加Trigger和JobDetail。

实体类：

```java
public class LeaveApplication {
    @TableId(type = IdType.AUTO)
    private Integer id;
    private Long proposerUsername;
    @JsonFormat( pattern = "yyyy-MM-dd HH:mm",timezone="GMT+8")
    private LocalDateTime startTime;
    @JsonFormat( pattern = "yyyy-MM-dd HH:mm",timezone="GMT+8")
    private LocalDateTime endTime;
    private String reason;
    private String state;
    private String disapprovedReason;
    private Long checkerUsername;
    private LocalDateTime checkTime;
}
```

Service层逻辑，重要的地方已在注释中说明。

### JobDetail和Trigger还有Scheduler

- [客户端类]()将[待执行的任务类]()封装成JobDetail和Trigger，然后将两者交给scheduler。这样JobDetail和Trigger都含有客户端类想要实现的逻辑和参数表，并且数据库也对其持久化
- 在封装JobDetail的同时，JobDetail会接收[客户端想要的Job类]()为入参，比如JobDetail接收LeaveStartJob.class。
  - 对于具体的业务场景，JobDetail接收LeaveStartJob.class这个操作会被执行多次，那么框架是如何识别该执行哪个JobDetail呢？通过withIdentity()方法，指定任务组名和任务名

```java
@Service
public class LeaveApplicationServiceImpl implements LeaveApplicationService {
    @Autowired
    private Scheduler scheduler;
  
       /**
     * 添加job和trigger到scheduler
     */
    private void addJobAndTrigger(LeaveApplication leaveApplication) {
        Long proposerUsername = leaveApplication.getProposerUsername();
        
        LocalDateTime startTime = leaveApplication.getStartTime();// 创建请假开始Job
        JobDetail startJobDetail = JobBuilder.newJob(LeaveStartJob.class)            
                .withIdentity(leaveApplication.getStartTime().toString(),
                        proposerUsername + "_start")// 指定任务组名和任务名        
                .usingJobData("username", proposerUsername)// 添加一些参数，执行的时候用
                .usingJobData("time", startTime.toString())
                .build();
        // 创建请假开始任务的触发器
        // 创建cron表达式指定任务执行的时间，由于请假时间是确定的，所以年月日时分秒都是确定的，这也符合任务只执行一次的要求。
        String startCron = String.format("%d %d %d %d %d ? %d",
                startTime.getSecond(),
                startTime.getMinute(),
                startTime.getHour(),
                startTime.getDayOfMonth(),
                startTime.getMonth().getValue(),
                startTime.getYear());
        CronTrigger startCronTrigger = TriggerBuilder.newTrigger()                
                .withIdentity(leaveApplication.getStartTime().toString(),
                        proposerUsername + "_start")// 指定触发器组名和触发器名
                .withSchedule(CronScheduleBuilder.cronSchedule(startCron))
                .build();

        // 将job和trigger添加到scheduler里
        try {
            scheduler.scheduleJob(startJobDetail, startCronTrigger);
        } catch (SchedulerException e) {
            e.printStackTrace();
            throw new CustomizedException("添加请假任务失败");
        }
    }
}
```

### QuartzJobBean（Job类逻辑）

```java
@Component
public class LeaveStartJob extends QuartzJobBean {
    private Scheduler scheduler;
    private SystemUserMapperPlus systemUserMapperPlus;

    @Autowired
    public LeaveStartJob(Scheduler scheduler,
                         SystemUserMapperPlus systemUserMapperPlus) {
        this.scheduler = scheduler;
        this.systemUserMapperPlus = systemUserMapperPlus;
    }

    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext)
            throws JobExecutionException {
        Trigger trigger = jobExecutionContext.getTrigger();
        JobDetail jobDetail = jobExecutionContext.getJobDetail();
        JobDataMap jobDataMap = jobDetail.getJobDataMap();
        // 将添加任务的时候存进去的数据拿出来
        long username = jobDataMap.getLongValue("username");
        LocalDateTime time = LocalDateTime.parse(jobDataMap.getString("time"));

        // 编写任务的逻辑

        // 执行之后删除任务
        try {
            // 暂停触发器的计时
            scheduler.pauseTrigger(trigger.getKey());
            // 移除触发器中的任务
            scheduler.unscheduleJob(trigger.getKey());
            // 删除任务
            scheduler.deleteJob(jobDetail.getKey());
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }
}
```

