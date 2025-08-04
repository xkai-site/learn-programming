# Quartz快速入门

Quartz适用于任务调度的Java框架，可以实现定时轮询的操作，常见业务有：定时更新/垃圾回收/训练模型等等。

本文学习自：[Quartz官方文档_w3cschool](https://www.w3cschool.cn/quartz_doc/)

-----

### 安装

如果主要是在应用的环境中使用quartz，所以一般将quartz jar包放到应用中（.ear或.war）。
如果你希望在很多应用中使用quartz，将quartz的jar包放在应用服务器(appserver)的classpath下即可。
如果你只是希望在独立的应用中使用quartz，将quartz的jar包和你的应用依赖的其它jar包放在一起即可。

quzrtz依赖一些第三方的库（以jar包的形式），这些库位于quartz安装包的lib目录下。要使用quartz的所有功能，必须将所有的第三方jar包都放到classpath下。如果你开发的是一个独立的quartz应用，建议将所有的jar包都放到classpath下；如果是在应用服务器环境下使用quartz，其中有些包可能已经存在于classpath中了，因此你需要自己选择。
==当然，我更喜欢Maven去做依赖管理==

-----

### 配置

quartz使用名为quartz.properties的配置文件。刚开始时该配置文件不是必须的，但是为了使用最基本的配置，该文件必须位于classpath下。

基于我的个人情况举个例子，我的应用是基于WebLogic Workshop开发的。我将所有的配置文件（包括quartz.properties）放到应用根目录下的一个项目中。当我将项目打包成.ear文件时，放置配置文件的项目会以jar包的形式进入最终的.ear包，所以quartz.properties文件就自动位于classpath中了。

如果你准备构建一个使用quartz的web应用（以.war包的形式），你应该将quartz.properties文件放到WEB-INF/classes目录下。

quartz是一个配置很灵活的应用。配置quartz最好的方式是，编辑quartz.properties文件，然后放到应用的classpath下。

quartz的安装包中包含了一些配置文件的示例，位于example/目录下。我建议你创建自己的quartz.properties文件，而不是简单地从示例中拷贝并删除不需要的部分。这样看起来更整洁，而且你也会了解到quartz的更多功能。

关于quartz配置文件的详细文档，请查阅[Quartz配置参考](https://www.w3cschool.cn/quartz_doc/quartz_doc-i7oc2d9l.html)

为了使用quartz，一个基本的quartz.properties配置文件如下所示：

```java
org.quartz.scheduler.instanceName = MyScheduler	//调度程序名
org.quartz.threadPool.threadCount = 3			//线程数
org.quartz.jobStore.class = org.quartz.simpl.RAMJobStore	//数据存储地址(RamJobStore表示内存存储)
```

---

### 核心

Job[是一个接口，需要实现类。调用时在JobDetail中绑定这个实现类]

```
void execute(JobExecutionContext context)
```

JobDetail具体调度程序

Trigger触发器，用于调度配置

Scheduler调度容器[可包含多个JobDetail与Trigger]

---

### 示例

```java
import org.quartz.*;
import org.quartz.impl.StdSchedulerFactory;

public class QuartzExample {
    public static void main(String[] args) throws SchedulerException {
        // 1. 获取调度器实例
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

        // 2. 定义 Job
        JobDetail job = JobBuilder.newJob(HelloJob.class)	//绑定Job的实现类
                .withIdentity("job1", "group1")	//此处的group1指的是Job的group名，与下面Trigger的group1没有关联
                .build();

        // 3. 定义 Trigger（触发器每 40 秒执行一次）
        Trigger trigger = TriggerBuilder.newTrigger()
                .withIdentity("trigger1", "group1")
                .startNow()
                .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                        .withIntervalInSeconds(40)
                        .repeatForever())
                .build();

        // 4. 调度 Job 和 Trigger
        scheduler.scheduleJob(job, trigger);

        // 5. 启动调度器
        scheduler.start();
    }

    // 定义一个简单的 Job
    public static class HelloJob implements Job {
        @Override
        public void execute(JobExecutionContext context) {
            System.out.println("Hello, Quartz! Current time: " + new java.util.Date());
        }
    }
}
```
