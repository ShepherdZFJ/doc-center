# quartz框架浅析

#### 1.什么是quartz

Quartz是OpenSymphony开源组织在Job scheduling领域又一个开源项目，完全由Java开发，可以用来执行定时任务，类似于java.util.Timer。但是相较于Timer， Quartz增加了很多功能，作为一个优秀的开源调度框架，Quartz具有以下特点：

- 强大的调度功能，例如支持丰富多样的调度方法，可以满足各种常规及特殊需求
- 灵活的应用方式，支持调度数据的多种存储方式
- 分布式和集群能力



#### 2.quartz框架的核心元素

Quartz调度依靠的三大核心元素就是：Scheduler、Trigger、Job。

- **Job(任务)**  ：具体要执行的业务逻辑，比如：发送短信、发送邮件、访问数据库、同步数据等
- **Trigger(触发器)**  ：用来定义Job（任务）触发条件、触发时间，触发间隔，终止时间等。有四大类型的触发器：
  - SimpleTrigger：简单触发器，从某个时间开始，每隔多少时间触发，重复多少次。
  - CornTrigger：使用cron表达式定义触发的时间规则
  - DateIntervalTrigger：每天中的一个时间段，每N个时间单元触发，时间单元可以是毫秒，秒，分，小时
- **scheduler(调度器)** ：Scheduler启动Trigger去执行Job，**Scheduler**由scheduler工厂创建：DirectSchedulerFactory或者StdSchedulerFactory。第二种工厂StdSchedulerFactory使用较多，因为DirectSchedulerFactory使用起来不够方便，需要作许多详细的手工编码设置。Scheduler主要有三种：RemoteMBeanScheduler，RemoteScheduler和StdScheduler。

Quartz核心元素之间的关系如下图所示：

<img src="/Users/shepherdmy/Desktop/WechatIMG458.png" alt="1111" style="zoom:50%;" />



#### 3.quartz集群原理分析

1)  jobStore的存储方式有两种:RAMJobStore, JobStoreSupport，其中RAMJobStore是将trigger和job存储在内存中，而JobStoreSupport是基于jdbc将trigger和job存储到数据库中。RAMJobStore的存取速度非常快，但是由于其在系统被停止后所有的数据都会丢失，所以在集群应用中，必须使用JobStoreSupport。

- qrtz_job_details表：记录每个任务的详细信息

  ```sql
  CREATE TABLE `qrtz_job_details` (
    `SCHED_NAME` varchar(120) COLLATE utf8_bin NOT NULL COMMENT '调度器名,集群环境中使用,必须使用同一个名称——集群环境下”逻辑”相同的scheduler,默认为QuartzScheduler',
    `JOB_NAME` varchar(200) COLLATE utf8_bin NOT NULL COMMENT '集群中job的名字',
    `JOB_GROUP` varchar(200) COLLATE utf8_bin NOT NULL COMMENT '集群中job的所属组的名字',
    `DESCRIPTION` varchar(250) COLLATE utf8_bin DEFAULT NULL COMMENT '描述',
    `JOB_CLASS_NAME` varchar(250) COLLATE utf8_bin NOT NULL COMMENT '集群中个note job实现类的完全包名,quartz就是根据这个路径到classpath找到该job类',
    `IS_DURABLE` varchar(1) COLLATE utf8_bin NOT NULL COMMENT '是否持久化,把该属性设置为1，quartz会把job持久化到数据库中',
    `IS_NONCONCURRENT` varchar(1) COLLATE utf8_bin NOT NULL COMMENT '是否并行，该属性可以通过注解配置',
    `IS_UPDATE_DATA` varchar(1) COLLATE utf8_bin NOT NULL,
    `REQUESTS_RECOVERY` varchar(1) COLLATE utf8_bin NOT NULL COMMENT '当一个scheduler失败后，其他实例可以发现那些执行失败的Jobs，若是1，那么该Job会被其他实例重新执行，否则对应的Job只能释放等待下次触发',
    `JOB_DATA` blob COMMENT '一个blob字段，存放持久化job对象',
    PRIMARY KEY (`SCHED_NAME`,`JOB_NAME`,`JOB_GROUP`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='存储每一个已配置的 Job 的详细信息';
  
  ```

- **qrtz_triggers表**：存放触发器的信息

  ```sql
  CREATE TABLE `qrtz_triggers` (
    `SCHED_NAME` varchar(120) COLLATE utf8_bin NOT NULL COMMENT '调度器名，和配置文件org.quartz.scheduler.instanceName保持一致',
    `TRIGGER_NAME` varchar(200) COLLATE utf8_bin NOT NULL COMMENT '触发器的名字',
    `TRIGGER_GROUP` varchar(200) COLLATE utf8_bin NOT NULL COMMENT '触发器所属组的名字',
    `JOB_NAME` varchar(200) COLLATE utf8_bin NOT NULL COMMENT 'qrtz_job_details表job_name的外键',
    `JOB_GROUP` varchar(200) COLLATE utf8_bin NOT NULL COMMENT 'qrtz_job_details表job_group的外键',
    `DESCRIPTION` varchar(250) COLLATE utf8_bin DEFAULT NULL COMMENT '描述',
    `NEXT_FIRE_TIME` bigint(13) DEFAULT NULL COMMENT '下一次触发时间',
    `PREV_FIRE_TIME` bigint(13) DEFAULT NULL COMMENT '上一次触发时间',
    `PRIORITY` int(11) DEFAULT NULL COMMENT '线程优先级',
    `TRIGGER_STATE` varchar(16) COLLATE utf8_bin NOT NULL COMMENT '当前trigger状态，设置为ACQUIRED,如果设置为WAITING,则job不会触发',
    `TRIGGER_TYPE` varchar(8) COLLATE utf8_bin NOT NULL COMMENT '触发器类型',
    `START_TIME` bigint(13) NOT NULL COMMENT '开始时间',
    `END_TIME` bigint(13) DEFAULT NULL COMMENT '结束时间',
    `CALENDAR_NAME` varchar(200) COLLATE utf8_bin DEFAULT NULL COMMENT '日历名称',
    `MISFIRE_INSTR` smallint(2) DEFAULT NULL COMMENT 'misfire处理规则,1代表【以当前时间为触发频率立刻触发一次，然后按照Cron频率依次执行】,
     2代表【不触发立即执行,等待下次Cron触发频率到达时刻开始按照Cron频率依次执行�】,
     -1代表【以错过的第一个频率时间立刻开始执行,重做错过的所有频率周期后，当下一次触发频率发生时间大于当前时间后，再按照正常的Cron频率依次执行】',
    `JOB_DATA` blob COMMENT 'JOB存储对象',
    PRIMARY KEY (`SCHED_NAME`,`TRIGGER_NAME`,`TRIGGER_GROUP`),
    KEY `SCHED_NAME` (`SCHED_NAME`,`JOB_NAME`,`JOB_GROUP`),
    CONSTRAINT `qrtz_triggers_ibfk_1` FOREIGN KEY (`SCHED_NAME`, `JOB_NAME`, `JOB_GROUP`) REFERENCES `qrtz_job_details` (`SCHED_NAME`, `JOB_NAME`, `JOB_GROUP`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='存储已配置的 Trigger 的信息';
  ```

- **qrtz_corn_triggers表**：存放cron类型的触发器

  ```sql
  CREATE TABLE `qrtz_cron_triggers` (
    `SCHED_NAME` varchar(120) COLLATE utf8_bin NOT NULL COMMENT '集群名',
    `TRIGGER_NAME` varchar(200) COLLATE utf8_bin NOT NULL COMMENT '调度器名,qrtz_triggers表trigger_name的外键',
    `TRIGGER_GROUP` varchar(200) COLLATE utf8_bin NOT NULL COMMENT 'qrtz_triggers表trigger_group的外键',
    `CRON_EXPRESSION` varchar(200) COLLATE utf8_bin NOT NULL COMMENT 'cron表达式',
    `TIME_ZONE_ID` varchar(80) COLLATE utf8_bin DEFAULT NULL COMMENT '时区ID',
    PRIMARY KEY (`SCHED_NAME`,`TRIGGER_NAME`,`TRIGGER_GROUP`),
    CONSTRAINT `qrtz_cron_triggers_ibfk_1` FOREIGN KEY (`SCHED_NAME`, `TRIGGER_NAME`, `TRIGGER_GROUP`) REFERENCES `qrtz_triggers` (`SCHED_NAME`, `TRIGGER_NAME`, `TRIGGER_GROUP`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='存放cron类型的触发器';
  ```

- **qrtz_scheduler_state表**：存储所有节点的scheduler，会定期检查scheduler是否失效，记录了最后最新的检查时间，在quartz.properties中设置了CHECKIN_INTERVAL为1000，也就是每秒检查一次

  ```sql
  CREATE TABLE `qrtz_scheduler_state` (
    `SCHED_NAME` varchar(120) COLLATE utf8_bin NOT NULL COMMENT '调度器名称，集群名',
    `INSTANCE_NAME` varchar(200) COLLATE utf8_bin NOT NULL COMMENT '集群中实例ID，配置文件中org.quartz.scheduler.instanceId的配置',
    `LAST_CHECKIN_TIME` bigint(13) NOT NULL COMMENT '上次检查时间',
    `CHECKIN_INTERVAL` bigint(13) NOT NULL COMMENT '检查时间间隔',
    PRIMARY KEY (`SCHED_NAME`,`INSTANCE_NAME`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='调度器状态';
  ```

- **qrtz_fired_triggers表**：存储与已触发的Trigger相关的状态信息，以及相联Job的执行信息

  ```sql
  CREATE TABLE `qrtz_fired_triggers` (
    `SCHED_NAME` varchar(120) COLLATE utf8_bin NOT NULL COMMENT '调度器名称，集群名',
    `ENTRY_ID` varchar(95) COLLATE utf8_bin NOT NULL COMMENT '运行Id',
    `TRIGGER_NAME` varchar(200) COLLATE utf8_bin NOT NULL COMMENT '触发器名',
    `TRIGGER_GROUP` varchar(200) COLLATE utf8_bin NOT NULL COMMENT '触发器组',
    `INSTANCE_NAME` varchar(200) COLLATE utf8_bin NOT NULL COMMENT '集群中实例ID',
    `FIRED_TIME` bigint(13) NOT NULL COMMENT '触发时间',
    `SCHED_TIME` bigint(13) NOT NULL,
    `PRIORITY` int(11) NOT NULL COMMENT '线程优先级',
    `STATE` varchar(16) COLLATE utf8_bin NOT NULL COMMENT '状态',
    `JOB_NAME` varchar(200) COLLATE utf8_bin DEFAULT NULL COMMENT '任务名',
    `JOB_GROUP` varchar(200) COLLATE utf8_bin DEFAULT NULL COMMENT '任务组',
    `IS_NONCONCURRENT` varchar(1) COLLATE utf8_bin DEFAULT NULL COMMENT '是否并行',
    `REQUESTS_RECOVERY` varchar(1) COLLATE utf8_bin DEFAULT NULL COMMENT '是否恢复',
    PRIMARY KEY (`SCHED_NAME`,`ENTRY_ID`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='存储与已触发的 Trigger 相关的状态信息，以及相联 Job 的执行信息';
  ```

- **qrtz_locks表**：存储程序的悲观锁的信息。Quartz提供的锁表，为多个节点调度提供分布式锁，实现分布式调度，默认有2个锁：

  - STATE_ACCESS主要用在scheduler定期检查是否有效的时候，保证只有一个节点去处理已经失效的scheduler。
  - TRIGGER_ACCESS主要用在TRIGGER被调度的时候，保证只有一个节点去执行调度。

  ```sql
  CREATE TABLE qrtz_locks
  (
    SCHED_NAME VARCHAR2(120) NOT NULL,
    LOCK_NAME  VARCHAR2(40) NOT NULL,
    CONSTRAINT QRTZ_LOCKS_PK PRIMARY KEY (SCHED_NAME,LOCK_NAME)
  );
  ```

2)  quartz的集群模式指的是一个集群下多个节点管理同一批任务的调度，通过共享数据库的方式实现，保证同一个任务到达触发时间的时候，只有一台机器去执行该任务。每个节点部署一个单独的quartz实例，相互之间没有直接数据通信。

在Quartz中，有两类线程，Scheduler调度线程和任务执行线程：

- 任务执行线程：通常使用一个线程池(SimpleThreadPool)维护一组线程，负责实际每个job的执行
- Scheduler调度线程QuartzSchedulerThread ：轮询存储的所有 trigger，如果有需要触发的 trigger，即到达了下一次触发的时间，则从任务执行线程池获取一个空闲线程，执行与该 trigger 关联的任务。

quartz集群基于数据库锁的同步操作流程如下图所示：

![333](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/05dac2dd.png)

一个调度器实例在执行涉及到分布式问题的数据库操作前，首先要获取QUARTZ_LOCKS表中对应的行级锁，获取锁后即可执行其他表中的数据库操作，随着操作事务的提交，行级锁被释放，供其他调度实例获取。集群中的每一个调度器实例都遵循这样一种严格的操作规程。每当要进行与某种业务相关的数据库操作时，先去QRTZ_LOCKS表中查询操作相关的业务对象所需要的锁，在select语句之后加for update来实现。例如，TRIGGER_ACCESS表示对任务触发器相关的信息进行修改、删除操作时所需要获得的锁，sql如下所示：

```sql
select * from QRTZ_LOCKS where sched_name = ? and lock_name = ? for update
```

当一个线程使用上述的SQL对表中的数据执行查询操作时，若查询结果中包含相关的行，数据库就对该行进行ROW LOCK；若此时，另外一个线程使用相同的SQL对表的数据进行查询，由于查询出的数据行已经被数据库锁住了，此时这个线程就只能等待，直到拥有该行锁的线程完成了相关的业务操作，执行了commit动作后，数据库才会释放了相关行的锁，这个线程才能继续执行