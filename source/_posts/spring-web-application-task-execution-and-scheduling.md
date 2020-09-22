---
title: 스프링 웹 애플리케이션 - 태스크 수행 및 스케줄링
date: 2020-09-14
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

## Task Execution and Scheduling
스프링 프레임워크는 비동기적으로 어떠한 작업을 수행하거나 반복적으로 수행해야하는 스케줄링 기능을 지원하기 위하여 `TaskExecutor`와 `TaskScheduler` 인터페이스를 제공합니다. 

### Scheduling
스프링 프레임워크에서 반복적으로 수행하는 스케줄링은 `TaskScheduler` 인터페이스에 의해 동작합니다. 그리고 스케줄링 기능을 활성화하기 위한 `@EnableScheduling` 어노테이션을 제공합니다.

구성 메타정보 클래스에 @EnableScheduling을 선언하면 `@Scheduled`에 의한 스케줄링을 활성화 할 수 있습니다. @Scheduled가 선언된 스케줄링 핸들러 함수를 수행하는 주체는 `TaskExecutor`가 담당합니다.

일반적으로 사용되는 TaskScheduler는 `ThreadPoolTaskScheduler`입니다.
```java ScheduleConfig.java
@EnableScheduling
@ComponentScan({"com.example.demo.schedule"})
@Configuration
public class ScheduleConfig {
    @Bean
    public TaskScheduler threadPoolTaskScheduler() {
        ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        taskScheduler.setPoolSize(10);
        return taskScheduler;
    }
}
```

예를 들어, 다음과 같이 크론 표현식을 사용하여 `1분 마다` 동작하는 스케줄링을 추가할 수 있습니다.

```java
@Component
public class ExampleScheduler {
    private static final Logger LOG = LoggerFactory.getLogger(ExampleScheduler.class);
    
    public ExampleScheduler() {}

    @Scheduled(cron = "0 * * * * *", zone = "Asia/Seoul")
    public void schedule() {
       LOG.info("run, {}", new Date());
    }
}
```

### Asynchronous Execution
스프링 프레임워크는 태스크에 대한 수행을 `TaskExecutor` 인터페이스가 담당합니다. 그리고 비동기로 수행하는 것을 지원하기 위하여 `@EnableAsync`와 `@Async` 어노테이션을 제공합니다.

구성 메타정보 클래스에 `@EnableAsync`를 선언하면 `@Async`가 선언된 함수는 비동기로 수행되며 수행을 담당하는 주체는 `TaskExecutor`가 됩니다.

일반적으로 많이 사용되는 TaskExecutor 구현체는 `ThreadPoolTaskExecutor`입니다. 
```java
@EnableAsync
@ComponentScan({"com.example.demo.handler"})
@Configuration
public class AsyncConfig implements AsyncConfigurer {

    @Bean
    public TaskExecutor asyncThreadPoolTaskExecutor() {
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(5);
        taskExecutor.setMaxPoolSize(10);
        taskExecutor.setQueueCapacity(25);
        return taskExecutor;
    }

    @Override
    public Executor getAsyncExecutor() {
        return asyncThreadPoolTaskExecutor();
    }
}
```

위와 같이 `AsyncConfigurer` 인터페이스를 통해 비동기로 수행되는 `TaskExecutor`를 변경할 수 있게 되어 asyncThreadPoolTaskExecutor에 의해 스레드 풀에서 반환된 TaskExecutor가 비동기로 수행합니다.

```java
@Service
public class ExampleService {
    private static final Logger LOG = LoggerFactory.getLogger(ExampleService.class);

    public ExampleService() {}

    @Async
    public void execute() {
        LOG.debug("executed");
    }
}
```

## Quartz Scheduler
[`Quartz`](http://www.quartz-scheduler.org/)는 자바 애플리케이션을 위한 스케줄러 라이브러리입니다. 스프링의 태스크 스케줄러는 단일 애플리케이션에는 문제없이 사용할 수 있지만 분산 환경에서 동작해야하는 웹 애플리케이션에서는 사용할 수 없습니다. 이와 비교하여 쿼츠 스케줄러는 메모리 기반의 스케줄링 뿐만 아니라 JTA 트랜잭션과 클러스터링을 지원합니다.

`quartz`와 `spring-tx` 의존성을 추가합니다.
```groovy build.gradle
implementation 'org.quartz-scheduler:quartz:2.3.2'
implementation 'org.quartz-scheduler:quartz-jobs:2.3.2'
implementation 'org.springframework:spring-tx:5.2.8.RELEASE'
implementation 'org.springframework:spring-context-support:5.2.8.RELEASE'
```

클래스패스에 `quartz.properties`를 생성하여 쿼츠 스케줄러에서 사용되는 프로퍼티를 기술합니다.

```properties quartz.properties
#============================================================================
# Configure Main Scheduler Properties
#============================================================================

org.quartz.scheduler.instanceName = MyClusteredScheduler
org.quartz.scheduler.instanceId = AUTO

#============================================================================
# Configure ThreadPool
#============================================================================

org.quartz.threadPool.class = org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount = 25
org.quartz.threadPool.threadPriority = 5
```

application.properties에 쿼츠 스케줄러 설정을 위한 프로퍼티를 기술합니다.
```properties application.properties
spring.quartz.job-store-type=MEMORY
spring.quartz.scheduler-name=quartz-scheduler
spring.quartz.start-up-delay=0
spring.quartz.auto-startup=true
spring.quartz.overwrite-existing-jobs=true
spring.quartz.wait-for-jobs-to-complete-on-shutdown=true
```

### SpringBeanJobFactory
`spring-context-support` 모듈에는 스케줄을 수행하는 `QuartzJobBean`을 동적으로 생성시킬 수 있도록 지원하는 `SpringBeanJobFactory` 클래스를 제공합니다.

```java
@EnableScheduling
@ComponentScan({"com.example.demo.schedule"})
@Configuration
public class ScheduleConfig {
    @Bean
    public JobFactory quartzJobFactory(AutowireCapableBeanFactory beanFactory) {
        return new SpringBeanJobFactory(){
            @Override
            protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
                Object job = super.createJobInstance(bundle);
                beanFactory.autowireBean(job);
                return job;
            }
        };
    }
}
```

### SchedulerFactoryBean
`spring-context-support` 모듈에는 쿼츠 스케줄러에 대한 라이프사이클을 관리할 수 있도록 `SchedulerFactoryBean` 클래스를 제공합니다. 

다음과 같이 SchedulerFactoryBean를 빈으로 등록합니다.
```java
@EnableScheduling
@ComponentScan({"com.example.demo.schedule"})
@Configuration
public class ScheduleConfig implements ApplicationContextAware {

    private ApplicationContext applicationContext;
    
    @Bean
    public JobFactory quartzJobFactory(AutowireCapableBeanFactory beanFactory) {
        return new SpringBeanJobFactory(){
            @Override
            protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {
                Object job = super.createJobInstance(bundle);
                beanFactory.autowireBean(job);
                return job;
            }
        };
    }

    @Bean
    public TaskExecutor quartzThreadPoolTaskExecutor() {
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(5);
        taskExecutor.setMaxPoolSize(10);
        taskExecutor.setQueueCapacity(25);
        return taskExecutor;
    }

    @Bean
    public SchedulerFactoryBean schedulerFactoryBean(JobFactory jobFactory, List<Trigger> triggers) throws IOException {
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        schedulerFactoryBean.setApplicationContext(applicationContext);
        schedulerFactoryBean.setTaskExecutor(quartzThreadPoolTaskExecutor());
        schedulerFactoryBean.setJobFactory(jobFactory);
        schedulerFactoryBean.setTriggers(triggers.toArray(new Trigger[triggers.size()]));

        Environment environment = applicationContext.getEnvironment();
        QuartzProperties quartzProperties = new QuartzProperties(environment);
        schedulerFactoryBean.setSchedulerName(quartzProperties.getSchedulerName());
        schedulerFactoryBean.setStartupDelay(quartzProperties.getStartUpDelay());
        schedulerFactoryBean.setAutoStartup(quartzProperties.isAutoStartup());
        schedulerFactoryBean.setOverwriteExistingJobs(quartzProperties.isOverwriteExistingJobs());
        schedulerFactoryBean.setWaitForJobsToCompleteOnShutdown(quartzProperties.isWaitForJobsToCompleteOnShutdown());
        schedulerFactoryBean.setQuartzProperties(quartzProperties.getProperties());
        return schedulerFactoryBean;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

위 예제 코드에서 QuartzProperties는 다음과 같이 구성하였습니다.

```java
public enum JobStoreType {
    MEMORY, JDBC
}

@Component
public class QuartzProperties {

    private JobStoreType jobStoreType;
    private String schedulerName;
    private int startUpDelay;
    private boolean autoStartup;
    private boolean overwriteExistingJobs;
    private boolean waitForJobsToCompleteOnShutdown;
    private Properties properties;

    public QuartzProperties(Environment environment) throws IOException {
        this.jobStoreType = environment.getProperty("spring.quartz.job-store-type", JobStoreType.class, JobStoreType.MEMORY);
        this.schedulerName = environment.getProperty("spring.quartz.scheduler-name", String.class, "quartz-scheduler");
        this.startUpDelay = environment.getProperty("spring.quartz.start-up-delay", Integer.class, 0);
        this.autoStartup = environment.getProperty("spring.quartz.auto-startup", Boolean.class, false);
        this.overwriteExistingJobs = environment.getProperty("spring.quartz.overwrite-existing-jobs", Boolean.class, false);
        this.waitForJobsToCompleteOnShutdown = environment.getProperty("spring.quartz.wait-for-jobs-to-complete-on-shutdown", Boolean.class, false);

        PropertiesFactoryBean propertiesFactoryBean = new PropertiesFactoryBean();
        propertiesFactoryBean.setLocation(new ClassPathResource("quartz.properties"));
        propertiesFactoryBean.afterPropertiesSet();
        this.properties = propertiesFactoryBean.getObject();
    }

    // getters...
}
```

### QuartzJobBean
`spring-context-support`에는 쿼츠 스케줄러가 스케줄을 수행하는 Job 구현체를 제공합니다. 애플리케이션 개발자는 QuartzJobBean를 빈으로 등록함으로써 쿼츠 스케줄러 잡을 등록할 수 있습니다.

다음은 QuartzJobBean을 확장한 잡을 구성하는 예제입니다.
```java
@Service
public class QuartzSampleJob extends QuartzJobBean {

    private static final Logger LOG = LoggerFactory.getLogger(QuartzSampleJob.class);
    private static final String IDENTITY = "QuartzSampleJob";

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        LOG.info("Execute using quartz scheduler!");
    }

    @Bean(name = "QuartzSampleJob")
    public JobDetail jobDetail() {
        return JobBuilder.newJob().ofType(this.getClass())
                .withIdentity(IDENTITY)
                .build();
    }

    @Bean(name = "QuartzSampleJobTrigger")
    public CronTriggerFactoryBean jobTrigger(@Qualifier("QuartzSampleJob") JobDetail jobDetail) {
        CronTriggerFactoryBean factoryBean = new CronTriggerFactoryBean();
        factoryBean.setJobDetail(jobDetail);
        factoryBean.setCronExpression("0/5 * * * * ?");
        return factoryBean;
    }
}
```

쿼츠 스케줄러는 `JobDetail`을 통해 Job의 수행 방식과 실행에 필요한 정보를 참조합니다. 또한, Trigger에는 JobDetail을 어떻게 실행되어야하는지를 결정합니다. 위 예제에서는 CronTrigger를 사용하여 크론 표현식을 통해 반복적으로 수행하도록 하였습니다.

따라서, 쿼츠 스케줄러는 JobDetail과 Trigger에 의해 스케줄링 기능을 수행하며 애플리케이션 개발자인 우리는 QuartzJobBean에 대한 JobDetail과 Trigger를 빈으로 등록하는 과정을 수행하면 됩니다.

이제 애플리케이션을 수행하면 다음과 같이 쿼츠 스케줄러에 대한 로그와 함께 스케줄링이 수행되는 로그가 출력됩니다.

```
2020-09-18 15:54:40 [main] INFO  o.s.s.c.ThreadPoolTaskExecutor(181) - Initializing ExecutorService 'quartzThreadPoolTaskExecutor'
2020-09-18 15:54:40 [main] INFO  o.q.i.StdSchedulerFactory(1220) - Using default implementation for ThreadExecutor
2020-09-18 15:54:40 [main] INFO  o.q.c.SchedulerSignalerImpl(61) - Initialized Scheduler Signaller of type: class org.quartz.core.SchedulerSignalerImpl
2020-09-18 15:54:40 [main] INFO  o.q.core.QuartzScheduler(229) - Quartz Scheduler v.2.3.2 created.
2020-09-18 15:54:40 [main] INFO  o.q.simpl.RAMJobStore(155) - RAMJobStore initialized.
2020-09-18 15:54:40 [main] INFO  o.q.core.QuartzScheduler(294) - Scheduler meta-data: Quartz Scheduler (v2.3.2) 'quartz-scheduler' with instanceId 'NON_CLUSTERED'
  Scheduler class: 'org.quartz.core.QuartzScheduler' - running locally.
  NOT STARTED.
  Currently in standby mode.
  Number of jobs executed: 0
  Using thread pool 'org.quartz.simpl.SimpleThreadPool' - with 25 threads.
  Using job-store 'org.quartz.simpl.RAMJobStore' - which does not support persistence. and is not clustered.

2020-09-18 15:54:40 [main] INFO  o.q.i.StdSchedulerFactory(1374) - Quartz scheduler 'quartz-scheduler' initialized from an externally provided properties instance.
2020-09-18 15:54:40 [main] INFO  o.q.i.StdSchedulerFactory(1378) - Quartz scheduler version: 2.3.2
2020-09-18 15:54:40 [main] INFO  o.q.core.QuartzScheduler(2293) - JobFactory set to: com.example.demo.config.ScheduleConfig$1@611ffa8d
2020-09-18 15:54:40 [main] INFO  o.s.s.q.SchedulerFactoryBean(727) - Starting Quartz Scheduler now
2020-09-18 15:54:40 [main] INFO  o.q.core.QuartzScheduler(547) - Scheduler quartz-scheduler_$_NON_CLUSTERED started.
2020-09-18 15:54:40 [quartz-scheduler_Worker-1] INFO  c.e.d.s.QuartzSampleJob(21) - Execute using quartz scheduler!
```

### DisallowConcurrentExecution
쿼츠 스케줄러에는 스케줄이 동작하고 있는 Job이 수행되고있는 시간이 길어지면 다음에 실행되어야하는 작업과 겹치지 않도록 `DisallowConcurrentExecution`을 제공합니다.

```java
@Service
@DisallowConcurrentExecution
public class QuartzSampleJob extends QuartzJobBean {}
```

## Scheduler Clustering
스프링 프레임워크 기반의 웹 애플리케이션이 분산 환경에서 동작하는 경우 하나의 인스턴스에서만 스케줄링이 수행되어야하는 요구사항이 있습니다. 쿼츠 스케줄러는 JDBC을 사용하여 스케줄러 동작을 클러스터링할 수 있도록 지원하는 JDBCJobStore를 제공합니다.

### JDBCJobStore
`JDBCJobStore`는 JDBC API를 통해 스케줄러를 관리할 수 있는 기능을 제공합니다. 본 예제에서는 PostgreSQL을 기반으로 설정하도록 하겠습니다.

먼저 `postgresql` 의존성을 추가합니다.
```groovy build.gradle
implementation 'org.postgresql:postgresql:42.2.16'
```

그리고 다음과 같이 쿼츠 스케줄러 프로퍼티를 기술합니다.
```properties quartz.properties
#============================================================================
# Configure JobStore
#============================================================================

org.quartz.jobStore.misfireThreshold = 60000

org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.PostgreSQLDelegate
org.quartz.jobStore.useProperties = false
org.quartz.jobStore.dataSource = quartzDS
org.quartz.jobStore.tablePrefix = QRTZ_

org.quartz.jobStore.isClustered = true
org.quartz.jobStore.clusterCheckinInterval = 20000

#============================================================================
# Configure Datasources
#============================================================================

org.quartz.dataSource.quartzDS.driver = org.postgresql.Driver
org.quartz.dataSource.quartzDS.URL = jdbc:postgresql://localhost:5432/quartz
org.quartz.dataSource.quartzDS.user = quartz
org.quartz.dataSource.quartzDS.password = quartz
```

쿼츠 스케줄러에서 사용할 DataSource는 `quartzDS`이며 PostgreSQL 드라이버를 사용하도록 설정하였습니다. 위 프로퍼티에 기술된 정보대로 PostgreSQL에 데이터베이스를 생성합니다.
```sql
CREATE USER quartz WITH ENCRYPTED PASSWORD 'quartz';
CREATE DATABASE quartz;
GRANT ALL ON DATABASE quartz TO quartz;
```

[`tables_postgres.sql`](https://github.com/quartz-scheduler/quartz/blob/master/quartz-core/src/main/resources/org/quartz/impl/jdbcjobstore/tables_postgres.sql)을 실행하여 JDBCJobStore에서 사용할 테이블 스키마를 생성합니다.

- qrtz_blob_triggers
- qrtz_calendars
- qrtz_cron_triggers
- qrtz_fired_triggers
- qrtz_job_details
- qrtz_locks
- qrtz_paused_trigger_grps
- qrtz_scheduler_state
- qrtz_simple_triggers
- qrtz_simprop_triggers
- qrtz_triggers

다음과 같이 JobDetail에 대해 `storeDurably`를 활성화합니다.
```java
@Service
@DisallowConcurrentExecution
public class QuartzSampleJob extends QuartzJobBean {

    @Bean(name = "QuartzSampleJob")
    public JobDetail jobDetail() {
        return JobBuilder.newJob().ofType(this.getClass())
                .withIdentity(IDENTITY)
                .storeDurably(true)
                .build();
    }
}
```

이제 애플리케이션을 실행하면 쿼츠 스케줄러가 데이터베이스에 저장된 정보에 의해 스케줄링이 실행됩니다.

---

이제 여러분은 스프링 프레임워크 기반의 애플리케이션에서 비동기로 수행되어야하는 로직을 처리할 수 있으며 반복적으로 수행되어야하는 스케줄링을 적용할 수 있게 되었습니다.

