# 概述

quartz 是一款企业级的任务调度框架。

# 基本使用

引入依赖

```xml
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.3.0</version>
</dependency>
```

启动一个默认调度器

```java
public static void main(String[] args) {
    try {
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        scheduler.start();
	    // 在这直接编写任务
        scheduler.shutdown();
    } catch (SchedulerException e) {
        e.printStackTrace();
    }
}
```

此时没有任何任务执行，我们现在来创建一个任务。

任务类 实现 `org.quartz.Job` 接口。

```java
@Slf4j
public class HelloJob implements Job {

    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        log.info("hello quartz");
    }
}
```

调度器 Scheduler 进行一个调度任务，由两部分构成

`JobDetail` 任务详情，指定了任务类，任务名称，任务组，和执行的任务的参数。

`Trigger` 触发器，指定了触发规则和调度规则

```java
public static void main(String[] args) {
    try {
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
        scheduler.start();
		// 创建任务详情
        JobDetail jobDetail = JobBuilder.newJob(HelloJob.class)
            .withIdentity("job1", "group1")
            .build();
		// 创建触发器
        Trigger trigger = TriggerBuilder.newTrigger()
            .withIdentity("trg1", "group1")
            .startNow()
            .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                          .withIntervalInSeconds(10)
                          .repeatForever())
            .build();
		// 进行调度
        scheduler.scheduleJob(jobDetail,trigger);

        //scheduler.shutdown();
    } catch (SchedulerException e) {
        e.printStackTrace();
    }
}
```

以上 `JobDetail`任务详情指定 `HelloJob` 为任务类，和任务组，任务名

触发器 `Trigger` 设置了触发器名称，触发器组，触发方式（立即启动），调度规则（每10s执行一次,不停止）

查看日志,每10s执行一次

![](img/q1.png)

# 配置文件

quartz的通过一个配置文件`quartz.properties`来配置quartz的配置，如果项目运行的类路径根目录没有`quartz.properties`文件的话，则使用默认的配置文件，默认的配置文件在源码的 `org/quartz/quartz.properties`下

如过配置了自定义的配置文件，默认的配置文件将完全失效，所以最好是copy一份过来，基于原有的修改。

默认的配置如下，原本默认properties文件的使用的`:`作为键值分隔符，可以修改`=`方便使用，同样可以读取。

```properties
# Default Properties file for use by StdSchedulerFactory
# to create a Quartz Scheduler Instance, if a different
# properties file is not explicitly specified.
#

org.quartz.scheduler.instanceName= DefaultQuartzScheduler
org.quartz.scheduler.rmi.export= false
org.quartz.scheduler.rmi.proxy= false
org.quartz.scheduler.wrapJobExecutionInUserTransaction= false

org.quartz.threadPool.class= org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount= 10
org.quartz.threadPool.threadPriority= 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread= true

org.quartz.jobStore.misfireThreshold= 60000

org.quartz.jobStore.class= org.quartz.simpl.RAMJobStore
```

在配置的开头的注释中，可以发现，这些配置用于构造`org.quartz.impl.StdSchedulerFactory`实例，我们所有可配置的字段都可以在这里面找到。

# 传递参数

创建任务详情时设置参数` JobDataMap`,也可以直接设置`JobData`，job 通过 `JobDetail`获取。

```java
JobDataMap jobDataMap=new JobDataMap();
jobDataMap.put("username","张三");
jobDataMap.put("age",12);

JobDetail jobDetail = JobBuilder.newJob(HelloJob.class)
    .withIdentity("job1", "group1")
    .usingJobData(jobDataMap)
    .usingJobData("address","北京")
    .build();
```

通过`JobExecutionContext`获取

```java
@Slf4j
public class HelloJob implements Job {
    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        JobDataMap jobDataMap = jobExecutionContext.getJobDetail().getJobDataMap();

        String username = jobDataMap.getString("username");
        int age = jobDataMap.getInt("age");
        String address = jobDataMap.getString("address");

        log.info("hello,{},{},{}",username,age,address);
    }
}
```

如果按照bean规范对Job进行属性设置，并存在 set/get 方法，则创建任务的时候`JobFactory`会自动对这些属性赋值

```java
@Slf4j
@Data
public class HelloJob implements Job {

    private String username;
    private Integer age;
    private String address;

    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        log.info("hello,{},{},{}",username,age,address);
    }
    
}
```

# 状态与并发

默认情况下，如果一个任务的执行时间，超过了调度的周期时间，则调度会另起一个线程，继续执行任务。

```java
@Slf4j
public class HelloJob implements Job {
    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        log.info("start");
        try {
            // 执行消耗设置为5s
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("end");
    }
}

// 调度周期设置为3s
Trigger trigger = TriggerBuilder.newTrigger()
    .withIdentity("trg1", "group1")
    .startNow()
    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                  .withIntervalInSeconds(3)
                  .repeatForever())
    .build();
```

以上的demo，当过了3s后，会再开一个任务，此时存在多个任务并发执行的场景

![](img/q2.png)

如果只希望同时只存在一个线程执行该job，通过注解`@DisallowConcurrentExecution` 标注。

```java
@Slf4j
@DisallowConcurrentExecution
public class HelloJob implements Job {
```

此时相当于加了同步,第二个任务会等待其他任务执行完才执行。

![](img/q3.png)

如果希望执行完一次任务后，下次任务执行更新后的数据，而不是一开始设置的数据，则需要

使用注解`@PersistJobDataAfterExecution`标注任务类后，数据则会更新。

通常与 `@DisallowConcurrentExecution` 一起使用，防止两个job线程同时修改数据，引发线程安全问题。

```java
@Slf4j
@Data
@PersistJobDataAfterExecution
public class HelloJob implements Job {
    private String username;
    private Integer age;
    private String address;
    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        log.info("hello,{},{},{}", username, age, address);
		// 执行完后，更新jobData数据
        JobDataMap jobDataMap = jobExecutionContext.getJobDetail().getJobDataMap();
        jobDataMap.put("age", ++age);
    }
}
```

得到

![](img/q4.png)

# Trigger

## 触发时间

可以设置 开始触发时间，和结束触发时间

```java
// 2020年9月4日，17点25分10s
Date startTime = DateBuilder.dateOf(17, 25, 10, 4, 9, 2020);
Date endTime = DateBuilder.dateOf(17, 26, 10, 4, 9, 2020);

Trigger trigger = TriggerBuilder.newTrigger()
    .withIdentity("trg1", "group1")
    .startAt(startTime)
    .endAt(endTime)
    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                  .withIntervalInSeconds(3)
                  .repeatForever())
    .build();
```

## SimpleTrigger

简单的触发器，可用定义一个任务，什么时候执行，执行几次。

ex: 不做任何设置，立即触发，只执行一次。可用通过以上的设置触发时间

```java
Trigger trigger =TriggerBuilder.newTrigger()
    .withIdentity("trg1", "group1")
    .build();
```

ex: 绑定一个调度规则，此时生成的对象为 SimpleTrigger ，

repeat 表示重复，interval 表示间隔 ，两者需配合设置。只设置间隔，不设置重复，将只执行一次。只设置重复，不设置间隔或间隔为0会报错。

```java
SimpleTrigger trigger = TriggerBuilder.newTrigger()
    .withIdentity("trg1", "group1")
    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                  .withIntervalInSeconds(3)
                  .repeatForever())
    .build();
```

以上 方法是 `withIntervalInSeconds` 表示每次执行间隔多少秒，类似的方法还有 `withIntervalInMilliseconds` 毫秒，`withIntervalInMinutes` 分钟，`withIntervalInHours` 小时。

`repeatForever()` 表示一直重复，直到结束时间。 还可以使用`withRepeatCount()` 指定重复次数。

```java
SimpleTrigger trigger = TriggerBuilder.newTrigger()
    .withIdentity("trg1", "group1")
    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                  .withIntervalInMinutes(1)
                  .withRepeatCount(3))
    .build();
```

以上代码，设置重复n次，总共会执行 n+1 次任务。

## CronTrigger

CronTrigger 是触发器的一种，通过一个 cron 表达式来定义任务如何执行。

使用 `CronScheduleBuilder` 定义，cron 表达式 是使用 七个子表达式组成，年为可选项 ，最少为六位。

完整cron表达式  为 `秒 分 时 当月天数 月 当周天数 年(可选)`

```java
CronTrigger trigger = TriggerBuilder.newTrigger()
    .withIdentity("trigger1", "group1")
    .withSchedule(CronScheduleBuilder.cronSchedule("0/5 * * * * ?"))
    .build();
```

每个字段都有一个有效值， 秒和分钟的值为 0-59  ，时的值为 0-23，当月天数的值为 1-31 

**但是**，这个月份的定义有点坑 

[官方文档](http://www.quartz-scheduler.org/documentation/quartz-2.3.0/tutorials/tutorial-lesson-06.html)上的说明是，定义为 0-11 ，或是使用 字符串简写,如 JAN,FEB等

![](img/q5.png)

而网上的一些杂贴，前一节说 0-11,后一节说 1-12 ，搞得很乱。

而实际使用时，如果把月份设置为 0 ,则会抛出异常，表示月份必须为 1-12

```
Exception in thread "main" java.lang.RuntimeException: CronExpression '0/5 * * * 0 ?' is invalid.
	at org.quartz.CronScheduleBuilder.cronSchedule(CronScheduleBuilder.java:111)
	at com.plf.quartz.App.main(App.java:28)
Caused by: java.text.ParseException: Month values must be between 1 and 12
```

设置为当前月，比如现在 九月，设置为 9 或 SEP ，任务刚好是执行的，设为其他则不执行。

查看官方github，确实是个issue，有对应的文档修复pr，但是并没有部署

![](img/q6.png)

**最终**,月份应该使用 1-12 来定义。

当周天数的有效值为 1-7 ，其中 1 为 星期s天 sunday ,2-7接着为 周一到周六 ，当周天数和当天月份不能同时设置，设置了其中一个，另一个需设置为 `?`

年份的有效值就为公历的年份，比如2020 ，作为可选配置，可以不填。

特殊写法：

`-` : 表示从有个值到某个值 ，比如  月份设置为 `MAY-SEP` 或 `5-9`  ，表示 五月到九月都生效。

`*` : 表示每一个，任何。比如，秒字段设置为`*` 表示每秒都会执行。

`,` : 表示集合，比如月份设置为 `SEP,DEC` ，则表示 九月和 十二月生效。还可以与`-`一起使用，比如`MAY-SEP,DEC` 表示五月到九月和十二月生效。

`/` : 表示增量，比如在秒字段设置为 `32/5` 表示从32秒开始执行第一次，隔5秒执行一次，下次执行为37秒，再下次为42秒，依次类推。

`?` : 该符号只能用于 ，当周天数和当月天数，表示 无指定值，这两个字段特殊值只能设置其中一个，不能同时设置，另一个需设置为 `?`

`L` : 可以用于 周天 和 月天 字段，用于月天表示 每月的最后一天，比如 1月31日，非闰年的2月28日。用于周天的话，则表示 7 或 SAT ，表示周六，如果L在一个数字后面表示，每月的最后一个星期几，如6L 或 FRIL ，表示每月最后一个周五。

`W` : 表示工作日

`#` : 表示工作日

# WEB

想要使 调度器 随web容器一起启动，有两种方式

方式1，在 `web.xml`添加一个监听器，可以随容器初始化调度器

```xml
<context-param>
    <param-name>quartz:config-file</param-name>
    <param-value>quartz.properties</param-value>
</context-param>
<context-param>
    <param-name>quartz:shutdown-on-unload</param-name>
    <param-value>true</param-value>
</context-param>
<context-param>
    <param-name>quartz:wait-on-shutdown</param-name>
    <param-value>false</param-value>
</context-param>
<context-param>
    <param-name>quartz:start-scheduler-on-load</param-name>
    <param-value>true</param-value>
</context-param>
<listener>
    <listener-class>
        org.quartz.ee.servlet.QuartzInitializerListener
    </listener-class>
</listener>
```

方式2，配置servlet

```xml
<servlet>
    <servlet-name>QuartzInitializer</servlet-name>
    <servlet-class>org.quartz.ee.servlet.QuartzInitializerServlet</servlet-class>
    <init-param>
        <param-name>shutdown-on-unload</param-name>
        <param-value>true</param-value>
    </init-param>
    <load-on-startup>2</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>QuartzInitializer</servlet-name>
    <url-pattern>/quartz</url-pattern>
</servlet-mapping>
```

两种方法都可以让调度器Scheduler随servlet容器启动,并可以通过一个key从`ServletContext`中来获取到`StdSchedulerFactory`的实例

在 `servlet`中获取 `scheduler` 实例 

```java
@Override
protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    ServletContext servletContext = req.getServletContext();
    StdSchedulerFactory quartzFactory = (StdSchedulerFactory) servletContext.getAttribute(QuartzInitializerListener.QUARTZ_FACTORY_KEY);
    try {
        Scheduler scheduler = quartzFactory.getScheduler();
    } catch (SchedulerException e) {
        e.printStackTrace();
    }
}
```

这个key默认为 `QuartzInitializerListener.QUARTZ_FACTORY_KEY` 也可以通过以下配置去自定义 key

```xml
<context-param>
    <param-name>quartz:servlet-context-factory-key</param-name>
    <param-value>quartzFactory</param-value>
</context-param>
```

如果想在 Job 中获取到 `ServletContext` ，需要指定一个配置key

```xml
<context-param>
    <param-name>quartz:scheduler-context-servlet-context-key</param-name>
    <param-value>servletContext</param-value>
</context-param>
```

然后Job中通过配置的key从`schedulerContext`获取到 `servletContext`

```java
@Slf4j
@Data
public class HelloJob implements Job {
    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        try {
            SchedulerContext schedulerContext = jobExecutionContext.getScheduler().getContext();
            ServletContext servletContext = (ServletContext) schedulerContext.get("servletContext");
            String contextPath = servletContext.getContextPath();
            log.info("hello,{}",contextPath);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }
}
```

# 持久化

默认情况下，任务的信息是以内存的形式，储存在当前进程的内存中，如果服务重启或者关闭，所以的任务信息都会丢失，但是优点是快，想要任务在重启后任能继续执行，就要持久化了。

1，持久化则需要使用数据库，quartz有对应的初始化数据库的sql脚本，找到对应的数据库的脚本，进行初始化。

脚本文件2.3.0之前需要下载官方发布的的distribution包，在`docs/dbTables`目录下，2.3.0及以后，脚本文件在源码中的`org.quartz.impl.jdbcjobstore` 包下，和`java`文件放在一起，用什么版本的quartz就找到对应版本的初始化脚本，不要去网上乱找。

找到对应的脚本，我们这里使用 `tables_mysql_innodb.sql` ，执行脚本，最好单独找个数据库，和业务的数据区分开。

2，修改我们的quartz核心配置文件

替换任务存储类 `JobStore` ，替换掉原来的内存存储，此时要做出一个选择

`JobStoreCMT` : 允许quartz与程序中的其他事务绑定。

`JobStoreTX`：不与其他事务绑定，常用选择。

```properties
#org.quartz.jobStore.class= org.quartz.simpl.RAMJobStore

#org.quartz.jobStore.class= org.quartz.impl.jdbcjobstore.JobStoreCMT
org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreTX
```

然后配置使用的数据库方言，表明使用那种数据库，通常最多使用标准方言`StdJDBCDelegate`，如`mysql`

```properties
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
```

配置表名前缀，不配置的话，使用默认值`QRTZ_`

```properties
org.quartz.jobStore.tablePrefix = QRTZ_
```

配置数据源，自定义一个数据源的名称，数据库的连接配置，前缀使用`org.quartz.impl.StdSchedulerFactory#PROP_DATASOURCE_PREFIX` 字段，中间为数据源名称，后缀为连接的各个配置如`org.quartz.impl.StdSchedulerFactory#PROP_DATASOURCE_DRIVER`

```properties
org.quartz.jobStore.dataSource = myDS

org.quartz.dataSource.myDS.driver = com.mysql.cj.jdbc.Driver
org.quartz.dataSource.myDS.URL = jdbc:mysql://192.168.24.130:3306/qrtz
org.quartz.dataSource.myDS.user = root
org.quartz.dataSource.myDS.password = 123456
```

3，基本配置已完成，此时启动项目，任务会被持久化到数据库中，再次重启后会发现，同样的任务（同一个identify）无法再次被加载，旧的任务从上次执行的地方继续执行

```java
org.quartz.ObjectAlreadyExistsException: Unable to store Job : 'group1.job1', because one already exists with this identification.
	at org.quartz.impl.jdbcjobstore.JobStoreSupport.storeJob(JobStoreSupport.java:1113)
	at org.quartz.impl.jdbcjobstore.JobStoreSupport$2.executeVoid(JobStoreSupport.java:1067)
	at 
2020-10-16 11:38:58.677  INFO --- [eduler_Worker-1] com.plf.quartz.job.HelloJob              : hello,张三,50,北京 
```

此时如果想对任务进行操作，通过scheduler实例进行

```java
// 删除任务
scheduler.deleteJob(new JobKey("job1","group1"));
// 重启任务
scheduler.resumeJob(new JobKey("job1","group1"));
// 暂停任务
scheduler.pauseJob(new JobKey("job1","group1"));
```

默认情况下 , `jobData` 的数据会存储在`QRTZ_JOB_DETAILS` 表的`JOB_DATA`字段以BLOB的形式存储，可以通过以下配置，修改为字符串的形式，但是修改之后 `jobData`只能设置字符串的属性，其实不方便。

```properties
org.quartz.jobStore.useProperties=true
```

默认情况会使用 c3p0 作为连接池 ，可以修改为 hikaricp

```properties
org.quartz.dataSource.myDS.provider=hikaricp
```

# 集群

[quartz集群](http://www.quartz-scheduler.org/documentation/quartz-2.3.0/tutorials/tutorial-lesson-11.html) 可以满足多节点执行任务，当集群环境下，只希望任务在一个节点执行，防止任务被多个节点重复执行，通过统一的调度管理，进行任务分发，实现负载均衡，当其中一个节点挂掉后，其他节点可以顶替继续执行任务，保证高可用，quartz的集群是基于数据的，不通节点不会直接感知其他节点，所以集群配置是要基于quartz持久化。

修改配置文件 ，设置 `org.quartz.jobStore.isClustered` 为 `true` 开启集群, 不通的节点，配置应该一致，但是允许不同的节点的修改线程池大小，及 每个集群节点必须指定唯一的集群实例id (instanceId) ，该值设置为 AUTO ，可以自动生成

```properties
# 开启集群
org.quartz.jobStore.isClustered=true
# 自动生成实例id
org.quartz.scheduler.instanceId=AUTO
```

启动一个实例时，日志显示`which supports persistence. and is clustered.` 支持持久化且是集群的，表示配置成功

```java
2020-10-21 17:12:28.573  INFO --- [           main] org.quartz.impl.jdbcjobstore.JobStoreTX  : Using db table-based data access locking (synchronization). 
2020-10-21 17:12:28.576  INFO --- [           main] org.quartz.impl.jdbcjobstore.JobStoreTX  : JobStoreTX initialized. 
2020-10-21 17:12:28.581  INFO --- [           main] org.quartz.core.QuartzScheduler          : Scheduler meta-data: Quartz Scheduler (v2.3.0) 'DefaultQuartzScheduler' with instanceId 'xxxx31211603271548542'
  Scheduler class: 'org.quartz.core.QuartzScheduler' - running locally.
  NOT STARTED.
  Currently in standby mode.
  Number of jobs executed: 0
  Using thread pool 'org.quartz.simpl.SimpleThreadPool' - with 10 threads.
  Using job-store 'org.quartz.impl.jdbcjobstore.JobStoreTX' - which supports persistence. and is clustered.
```

启动多个实例，关掉其中一个正在执行的实例，其他实例会察觉到，并继续支持任务。

```java
2020-10-21 17:39:30.176  INFO --- [_ClusterManager] org.quartz.impl.jdbcjobstore.JobStoreTX  : ClusterManager: detected 1 failed or restarted instances. 
2020-10-21 17:39:30.176  INFO --- [_ClusterManager] org.quartz.impl.jdbcjobstore.JobStoreTX  : ClusterManager: Scanning for instance "xxx31211603273091117"'s failed in-progress jobs. 
2020-10-21 17:39:30.182  INFO --- [_ClusterManager] org.quartz.impl.jdbcjobstore.JobStoreTX  : ClusterManager: ......Freed 1 acquired trigger(s). 
2020-10-21 17:39:30.234  INFO --- [eduler_Worker-1] com.plf.quartz.job.HelloJob              : hello,张三,12,北京 
```



# spring

spring 提供了整合 quartz 的方案 ，详见[spring文档](https://docs.spring.io/spring-framework/docs/5.2.4.RELEASE/spring-framework-reference/integration.html#scheduling-quartz) 

需要的依赖

```xml
<!--spring-quartz整合包-->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <version>5.2.8.RELEASE</version>
</dependency>
<!--spring核心-->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.8.RELEASE</version>
</dependency>
<!--quartz核心-->
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.3.0</version>
</dependency>
```

整合spring之后，调度器随spring容器一起启动，依旧是三步配置，配置 `JobDetail` ，`Trigger` ,`Scheduler` ，只是spring都封装成了`FactoryBean`模式

```xml
<!--jobDetail-->
<bean id="jobDetail" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
    <property name="group" value="group1"/>
    <property name="name" value="job1"/>
    <property name="jobClass" value="com.plf.qrtzsp.job.HelloJob"/>
    <property name="jobDataAsMap">
        <map>
            <entry key="username" value="张三"/>
        </map>
    </property>
</bean>
<!--trigger-->
<bean id="cronTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
    <property name="group" value="group1"/>
    <property name="name" value="trigger1"/>
    <property name="cronExpression" value="0/3 * * * * ?"/>
    <property name="jobDetail" ref="jobDetail"/>
</bean>
<!--scheduler-->
<bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="triggers">
        <list>
            <ref bean="cronTrigger"/>
        </list>
    </property>
</bean>
```

任务类还是之前任务类，不过此时会丢失一个特性，就是只能从`JobExecutionContext`获取任务参数，不再自动注入属性

```java
@Slf4j
@Data
public class HelloJob implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDataMap jobDataMap = context.getJobDetail().getJobDataMap();
        String username = jobDataMap.getString("username");
        log.info("hello,"+username);
    }
}
```

如果需要使用自动注入则需要继承`QuartzJobBean`，而不是实现 `Job`

```java
@Slf4j
@Data
public class HelloJob extends QuartzJobBean {

    private String username;

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        log.info("hello,usernameF:{}",username);
    }
}
```

这种方式也可以注入其他bean到`jobDataMap`

```xml
<bean id="jobDetail" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
    <property name="group" value="group1"/>
    <property name="name" value="job1"/>
    <property name="jobClass" value="com.plf.qrtzsp.job.HelloJob"/>
    <property name="jobDataAsMap">
        <map>
            <entry key="username" value="张三"/>
            <entry key="studentService"  value-ref="studentService"/>
        </map>
    </property>
</bean>

<bean id="studentService" class="com.plf.qrtzsp.service.StudentService"/>
```

job类中使用bean

```java
@Slf4j
@Data
public class HelloJob extends QuartzJobBean {

    private StudentService studentService;

    private String username;

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        log.info("hello,username:{}",username);
        studentService.study();
    }
}
```

此时调度器随容器启动

```java
ApplicationContext context=new ClassPathXmlApplicationContext("spring.xml");
```

## 持久化

方式1，使用`quartz.properties`

只需要依赖 `spring-tx`

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-tx</artifactId>
    <version>5.2.8.RELEASE</version>
</dependency>
```

和之前的持久化一样，需要配置`quartz.properties`

调度器选择，这里必须指定下配置文件路径

```xml
<bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
	<!--....-->
    <property name="configLocation" value="classpath:quartz.properties"/>
</bean>
```

方式2，spring注入数据源到`SchedulerFactoryBean` ，无需配置`quartz.properties`

除了 `spring-tx`还需依赖 `spring-jdbc`

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.2.8.RELEASE</version>
</dependency>
```

注入`dataSource`到 `scheduler`

```xml
<bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <!--省略-->
    <property name="dataSource" ref="dataSource"/>
</bean>

<bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
    <property name="jdbcUrl" value="jdbc:mysql://192.168.24.130:3306/qrtz"/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
    <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
</bean>
```

此时会自动使用`JobStoreCMT` 模式，使用`org.springframework.scheduling.quartz.LocalDataSourceJobStore`

## 集群

在sprign中，可以在SchedulerFactoryBean的配置中，使用`quartzProperties` 属性，代替`quartz.properties`

同样是开启集群，自动生成实例id

```xml
<bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
	<!--以上省略-->
    <property name="quartzProperties">
        <props>
            <prop key="org.quartz.scheduler.instanceId">AUTO</prop>
            <prop key="org.quartz.jobStore.isClustered">true</prop>
        </props>
    </property>
</bean>
```



## job注入bean

持久化后，会发现，bean不能持久化，所以不能用在jobDataMap中，这时需要使用spring提供的bean注入

需要注入bean的job，把所有的需要注入的bean，设置作为一个构造方法

```java
@Slf4j
@Data
@NoArgsConstructor
public class HelloJob extends QuartzJobBean {

    private StudentService studentService;

    private String username;

    public HelloJob(StudentService studentService) {
        this.studentService = studentService;
    }

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        JobDataMap jobDataMap = context.getJobDetail().getJobDataMap();
        String username1 = jobDataMap.getString("username");
        log.info("hello,usernameF:{},usernameP:{}",username,username1);
        log.info("service:{}",studentService);
        studentService.study();
    }
}
```

并设置 `SpringBeanJobFactory` 到 `SchedulerFactoryBean `中，该任务工厂可以把匹配的bean注入job的实例中，还有properties

```xml
<bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
	<!--省略-->
    <property name="jobFactory">
        <bean class="org.springframework.scheduling.quartz.SpringBeanJobFactory"/>
    </property>
</bean>
```

至于为什么只能支持构造注入，查看源码发现，是通过`AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR`策略决定的。

```java
@Override
protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
    Object job = (this.applicationContext != null ?
                  this.applicationContext.getAutowireCapableBeanFactory().createBean(
                      bundle.getJobDetail().getJobClass(), AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR, false) :
                  super.createJobInstance(bundle));
```

# springboot

springboot2.0集成quartz

引入依赖，引入该starter之后，会自动实例化 Scheduler 对象，加入到容器中。并能够自动加载，JobDetail 和 Trigger的bean。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

定义job类，实现`QuartzJobBean` ，sb2.0之后可以直接使用`@Autowired`注入bean

```java
@Data
public class HelloJob extends QuartzJobBean {

    @Autowired
    private HelloService helloService;

    private String name;

    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        helloService.hello(name);
    }
}
```

其中`HelloService` 注册为一个bean。

```java
@Slf4j
@Service
public class HelloService {
    public void hello(String name){
      log.info("hello,{}",name);
    }
}
```

1，固定任务

把JobDetail 和 Trigger 注册为 bean，即可被Scheduler发现，并自动调度。

```java
@Slf4j
@Configuration
public class QuartzConfig {

    // 需要调用下storeDurably()方法，让任务暂存
    @Bean
    public JobDetail helloJobDetail() {
        JobDataMap jobDataMap = new JobDataMap();
        jobDataMap.put("name", "张三");
        return JobBuilder
                .newJob(HelloJob.class)
                .usingJobData(jobDataMap)
                .storeDurably()
                .build();
    }

    @Bean
    public Trigger helloTrigger(JobDetail jobDetail) {
        CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule("0/3 * * * * ?");
        return TriggerBuilder
                .newTrigger()
                .withSchedule(cronScheduleBuilder)
                .forJob(jobDetail)
                .build();
    }
}
```

2,动态任务

在某个bean中，注入Scheduler的bean，手动创建任务并调度，也是quartz的主要功能，固定的任务用spring-task即可，quartz可以动态创建任务，比如页面上点击下，在后台就创建了一个任务，动态的选择执行的周期，而不是固定的配置。

```java
@Service
public class StartService implements InitializingBean {

    @Autowired
    private Scheduler scheduler;

    @Override
    public void afterPropertiesSet() throws Exception {
        JobDataMap jobDataMap = new JobDataMap();
        jobDataMap.put("name","中山");
        JobDetail jobDetail = JobBuilder
                .newJob(HelloJob.class)
                .usingJobData(jobDataMap)
                .build();

        Trigger trigger = TriggerBuilder
                .newTrigger()
                .withSchedule(CronScheduleBuilder.cronSchedule("0/5 * * * * ?"))
                .build();

        try {
            scheduler.scheduleJob(jobDetail,trigger);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }
}
```

## 持久化

springboot这边持久化会方便些，主要定义三个属性，

`spring.quartz.job-store-type` 可以选择 memory 或者 jdbc，当选择jdbc时，同时需要有可用的 DataSource 的bean，才能生效。

`spring.quartz.jdbc.initialize-schema` 初始化数据库的模式，always即每次都会初始化数据库，就是会把初始化脚本执行一遍，每次都执行，还有的选项是 embedded ，只有在使用内嵌数据库才初始化，第三个选项是never，不初始化数据库。

`spring.quartz.jdbc.schema` 初始化的数据库脚本，quartz2.3.0之前的版本需要把脚本复制进来，之后的版本直接在jar包路径内。

```yaml
spring:
  quartz:
    job-store-type: jdbc
    jdbc:
      initialize-schema: always
      schema: classpath:org/quartz/impl/jdbcjobstore/tables_mysql_innodb.sql
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql://192.168.24.130:3306/qrtz
    driver-class-name: com.mysql.cj.jdbc.Driver
```

然后定义一个 数据源 DataSource 

为了能让DataSource产生bean，和正常连接数据库，还需要导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```

此时持久化配置完成。

如果想让业务的数据源和 quartz的数据源分开，可以单独定义一个DataSource，并使用`@QuartzDataSource`标注这个bean

```java
@Bean
@QuartzDataSource
public DataSource quartzDataSource(){
    HikariDataSource dataSource = new HikariDataSource();
    dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
    dataSource.setJdbcUrl("jdbc:mysql://192.168.24.130:3306/qrtz");
    dataSource.setUsername("root");
    dataSource.setPassword("123456");
    return dataSource;
}
```

## 集群

集群下， 多个sheduler实例共享一个qrtz数据库，一个任务持久化一次，保证多个实例下只执行一次，多个scheduler进行负载分发任务

修改配置, `application.yml` 中，可以通过 `spring.quartz.properties` 指定一个map，代替原来的`quartz.propreties`

这里设置参数 `org.quartz.jobStore.isClustered` 为true ，即可开启集群模式，还需要指定下`org.quartz.scheduler.instanceName` 实例名称，多个节点公用同一个schduler实例名称，代表一个集群。每个节点需要设置一个`org.quartz.scheduler.instanceId` ，不同的节点需要不同的id，设置为`AUTO` 自动生成i

```yaml
spring:
  quartz:
    job-store-type: jdbc
    jdbc:
      initialize-schema: always
      schema: classpath:org/quartz/impl/jdbcjobstore/tables_mysql_innodb.sql
    properties:
      org:
        quartz:
          jobStore:
            isClustered: true
          scheduler:
            instanceId: AUTO
            instanceName: testQuartz1
```

当启动第二个节点时，实例会对节点进行任务分发。