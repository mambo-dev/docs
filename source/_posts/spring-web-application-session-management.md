---
title: 스프링 웹 애플리케이션 - 세션 관리
date: 2020-09-17
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

스프링 프레임워크는 사용자 세션 정보를 관리하기 위한 API와 구현체를 제공하기 위한 `Spring Session` 프로젝트를 적용할 수 있습니다. 

- spring-session-core
- spring-session-data-jdbc
- spring-session-data-redis

## Session Management
스프링 프레임워크는 `spring-session-core` 모듈을 통해 사용자의 세션 정보를 관리하기 위한 구현체 및 API를 제공합니다. 

> 스프링 세션 프로젝트를 적용하는 다양한 예제는 [Samples and Guides](https://docs.spring.io/spring-session/docs/2.3.0.RELEASE/reference/html5/#samples)를 참고하시기 바랍니다.

### springSessionRepositoryFilter
스프링 세션이 Session에 대하여 HttpSession으로 통합하기 위하여 `@EnableSpringHttpSession`를 구성 메타정보 클래스에 선언합니다. 구성 메타정보에 @EnableSpringHttpSession가 선언되면 `springSessionRepositoryFilter`라는 이름의 특별한 필터가 빈으로 등록됩니다.

```java
@EnableSpringHttpSession
@Configuration
public class SessionConfig {

    @Bean
    public MapSessionRepository sessionRepository() {
        return new MapSessionRepository(new ConcurrentHashMap<>());
    }
}
```

### Servlet Container Initialization
서블릿 컨테이너 초기화시 springSessionRepositoryFilter를 참조할 수 있도록 루트 애플리케이션 컨텍스트에 등록합니다.

```java
public class WebServletInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { AppConfig.class, HttpSessionConfig.class };
    }
    
    //...
}
```

그리고 서블릿 컨테이너가 모든 요청에 대하여 `springSessionRepositoryFilter` 필터를 적용할 수 있도록 `AbstractHttpSessionApplicationInitializer`를 클래스패스에 추가합니다.
```java
public class HttpSessionApplicationInitializer extends AbstractHttpSessionApplicationInitializer {
}
```

이제 브라우저를 통해 http://localhost:8080 에 접근하면 쿠키에 `SESSION`이라는 속성으로 세션 아이디 값이 부여될 것입니다.

## Session Clustering
스프링 프레임워크 기반의 웹 애플리케이션을 분산 환경에서 구동한다면 인증된 사용자의 세션 정보를 모든 애플리케이션 인스턴스에서 동일하게 사용해야합니다. 

스프링 세션 프로젝트는 분산 세션 관리를 위해 메모리 기반의 저장소인 Redis를 활용하여 세션을 관리할 수 있도록 `spring-session-data-redis` 모듈을 제공합니다.

먼저, [`spring-session-data-redis`](https://mvnrepository.com/artifact/org.springframework.session/spring-session-data-redis/2.3.0.RELEASE)와 [`lettuce-core`](https://mvnrepository.com/artifact/io.lettuce/lettuce-core/5.3.3.RELEASE) 의존성을 추가합니다.

```groovy build.gradle
implementation 'org.springframework.session:spring-session-data-redis:2.3.0.RELEASE'
implementation 'io.lettuce:lettuce-core:5.3.3.RELEASE'
```

application.properties에 Redis 연결 정보를 기술합니다.

```properties application.properties
spring.redis.host=localhost
spring.redis.password=
spring.redis.port=6379
```

구성 메타 정보 클래스에 `@EnableRedisHttpSession` 선언하면 RedisHttpSessionConfiguration가 추가되면서 `springSessionRepositoryFilter`라는 필터가 자동으로 빈으로 등록됩니다. 이와 함께 기본적으로 `RedisIndexedSessionRepository`와 `RedisTemplate` 그리고 `SessionCleanupConfiguration`등이 빈으로 등록됩니다.

```java
@PropertySource({"classpath:/application.properties"})
@EnableRedisHttpSession
@Configuration
public class SessionConfig implements EnvironmentAware {

    private Environment environment;

    @Bean
    public LettuceConnectionFactory connectionFactory() {
        String hostName = environment.getProperty("spring.redis.host", String.class, "localhost");
        String password = environment.getProperty("spring.redis.password", String.class);
        int port = environment.getProperty("spring.redis.port", Integer.class, 6379);

        LettuceConnectionFactory connectionFactory = new LettuceConnectionFactory();
        RedisStandaloneConfiguration standaloneConfiguration = connectionFactory.getStandaloneConfiguration();
        standaloneConfiguration.setHostName(hostName);
        standaloneConfiguration.setPassword(password);
        standaloneConfiguration.setPort(port);
        return connectionFactory;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
}
```

> Redis 저장소 접근을 위해 Lettuce를 사용하도록 `LettuceConnectionFactory`를 빈으로 등록하였습니다.

이제 웹 애플리케이션을 다시 실행하여도 종료하기전에 사용하던 세션 정보는 Redis에 저장되어있어 세션 정보를 유지할 수 있습니다.

---

이제 여러분은 애플리케이션 사용자의 세션을 관리할 수 있고 분산 환경의 애플리케이션인 경우 Redis와 같은 저장소를 활용하여 관리할 수 있음을 알게되었습니다.