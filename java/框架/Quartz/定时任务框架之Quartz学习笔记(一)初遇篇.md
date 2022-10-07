# Quartz学习笔记(一) 初遇篇

> 我工作日常与定时任务打交道还是满频繁的，我们用的是分布式定时任务自己的一套实现,  但也不好一开始直接过度到分布式定时任务，我们就先从单机的定时任务框架学起。本次选取的定时任务框架是Java领域内比较知名的Quartz框架。

[TOC]

## 该怎么让这个方法定时执行

本篇我们的诉求就只有一个，怎么让我们的方法在指定的时间点开始执行。首先我们引入Quartz的依赖：

```xml
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.3.2</version>
</dependency>
```

下面是Quartz的示例

```java
public class HelloJob implements Job {
    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        JobDetail jobTetail = jobExecutionContext.getJobDetail();
        System.out.println("hello world ------");
    }
}
public static void main(String[] args) throws SchedulerException {
        SchedulerFactory schedFact = new StdSchedulerFactory();
        Scheduler sched = schedFact.getScheduler();
        sched.start();
        // define the job and tie it to our HelloJob class
        JobDetail job = newJob(HelloJob.class)
                .withIdentity("myJob", "group1")
                .build();

        // Trigger the job to run now, and then every 40 seconds
        Trigger trigger = newTrigger()
                .withIdentity("myTrigger", "group1") // 将该触发器放入到分组中
                .startNow() // 现在就开始了
                .withSchedule(simpleSchedule()
                        .withIntervalInSeconds(1) // 执行频率
                        .repeatForever()) // 重复执行
                .build();
        // Tell quartz to schedule the job using our trigger
        sched.scheduleJob(job, trigger);
  }
```

如上图所示我们就实现了HelloJob中的execute方法每秒执行一次。我们大致通过上面一个简单的例子来看在Quartz的一些基本概念:

![定时任务框架Quartz架构](http://tvax2.sinaimg.cn/large/006e5UvNgy1h2g7ng28mfj30qe0ermyl.jpg)

触发器定时任务这个定时的逻辑: 

- 在特定时间之后开始还是现在就开始触发
- withSchedule定义执行频率、执行次数。

似乎大致可以理解为调度器每时每刻都在轮询触发器的条件是否满足，如果满足实现HelloJob中的方法则会被执行。这是很直观明了的框架图，Quartz为定时任务又添加了一些描述性信息: 

![更进一步的任务调度](http://tva4.sinaimg.cn/large/006e5UvNgy1h2g84hnz5rj312a0kr0xf.jpg)

到现在其实已经满足了我们的问题了, 但是我们又不仅仅满足于此，我们想要定时任务按cron表达式(cron表达式来指定任务在某个时间点或者周期性的执行)执行呢,simpleSchedule是SimpleScheduleBuilder的静态方法, 我们大致看一下SimpleScheduleBuilder的基本结构:

![SimpleScheduleBuilder的组成](http://tvax1.sinaimg.cn/large/006e5UvNgy1h2g8cc7nncj30nl099428.jpg)



那有没有这样一种可能在Quartz的所有调度器的构建都是ScheduleBuilder的子类呢？

![scheduleBuilder继承图](http://tva1.sinaimg.cn/large/006e5UvNgy1h2g8fn9q9hj30g80cc40c.jpg)



我们来浅用一下:

```java
public static void main(String[] args) throws SchedulerException {
    SchedulerFactory schedFact = new StdSchedulerFactory();
    Scheduler sched = schedFact.getScheduler();
    sched.start();
    // define the job and tie it to our HelloJob class
    JobDetail job = newJob(HelloJob.class)
            .withIdentity("myJob", "group1")
            .build();

    // Trigger the job to run now, and then every 40 seconds
    Trigger trigger = newTrigger()
            .withIdentity("myTrigger", "group1")
            .startNow()
            .withSchedule(cronSchedule("")) // 填cron表达式即可
            .build();

    // Tell quartz to schedule the job using our trigger
    sched.scheduleJob(job, trigger);
}
```

下一步我们看Quartz与Spring Boot整合。

## Quartz整合Spring Boot

还记得Spring Boot 整合 其他框架的步骤吗？ 首先找starter，我们尽量不看其他博客，看看我们只看官方文档是否能完成整合：

![Spring Boot 整合](http://tva1.sinaimg.cn/large/006e5UvNgy1h2g8wq7unnj313l0lgqh0.jpg)



![SpringBoot整合(二)](http://tvax1.sinaimg.cn/large/006e5UvNgy1h2g8xy0wd4j30sh0kegsv.jpg)

​	![Quartz整合](http://tvax4.sinaimg.cn/large/006e5UvNgy1h2g8yxbbzjj30y60ojnfz.jpg)

![SpringBoot 整合Quartz](http://tvax1.sinaimg.cn/large/006e5UvNgy1h2g90iaz8tj30v50p9dvv.jpg)

​	我们这次只取我们所需要的，从上面的描述我们可以看出Scheduler已经被Spring构建了，我们只用定义触发器、任务就能自动关联调度器。官方给的示例是实现QuartzJobBe

```java
@Component
public class HelloJob extends QuartzJobBean {
    // 在这里面写定时任务的处理逻辑
    // 然后我们将JobDetail和Trigger加入到IOC容器中
    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        System.out.println(" hello world");
    }
}

@Configuration
public class JobConfiguration {

    @Value("${cron:*/5 * * * * ?}")
    private String valueCron;

    @Bean
    public JobDetail  buildJobDetail(){
        JobDetail jobDetail = JobBuilder.newJob(HelloJob.class).withIdentity("唯一").storeDurably().build();
        return jobDetail;
    }
    @Bean
    public Trigger buildTrigger(){
        Trigger trigger = newTrigger()
                .withIdentity("myTrigger", "group1")
                .startNow()
                .withSchedule(cronSchedule(valueCron)).forJob("唯一") // 关联任务和触发器
                .build();
        return trigger;
    }
}
```

## 延迟执行

有的时候我们的要求可能不是那么高，不值得动用Quartz出马，比如我们想要延迟执行, 且执行一次，也不用如此大动干戈。JDK 内置的就有这样的轻量级定时任务组件：

```java
private static final  ScheduledExecutorService scheduledExecutorService  = Executors.newScheduledThreadPool(10);
// 延迟一秒再执行
public void studySchedule(){
    scheduledExecutorService.schedule(()->{
        System.out.println("hello world");
    },1,TimeUnit.SECONDS);
}
```

## 参考资料

- 简洁明了看懂cron表达式 https://zhuanlan.zhihu.com/p/437328366