---
title: 스프링 웹 애플리케이션 - 태스크 수행 및 스케줄링
date: 2020-09-14
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
description: 잠만보와 함께하는 스프링 프레임워크 기반의 웹 애플리케이션
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

> 업데이트 대기중입니다.

---

이제 여러분은 스프링 프레임워크 기반의 애플리케이션에서 비동기로 수행되어야하는 로직을 처리할 수 있으며 반복적으로 수행되어야하는 스케줄링을 적용할 수 있게 되었습니다.

