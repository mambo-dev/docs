---
title: 스프링 웹 애플리케이션 - 데이터베이스 액세스
date: 2020-09-14
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

일반적으로 웹 애플리케이션에서 발생하는 데이터를 저장하기 위하여 관계형 데이터베이스를 사용합니다. 스프링 프레임워크는 관계형 데이터베이스에 연결할 수 있도록 DataSource에 대한 추상화를 제공합니다. 따라서, JDBC 또는 JPA와 같은 데이터 액세스 기술을 적용할 수 있도록 지원하고 있습니다.

## Data Access
스프링 프레임워크는 JDBC, JPA와 같은 `데이터 액세스` 기술과 함께 `Data Access Object`를 지원합니다. 스프링 프레임워크에서 제공하는 `@Repository`는 DAO를 지칭할 수 있는 가장 좋은 방법입니다. 스프링 프레임워크는 `@Repository`가 선언된 빈 클래스에서 발생되는 `SQLException`을 일관된 `DataAccessException`으로 변환하는 작업을 수행합니다.

### DataSource
스프링은 `DataSource` 인터페이스를 통해 데이터베이스에 대한 `커넥션`을 가져옵니다. 애플리케이션 개발자는 데이터베이스에 어떤 방식으로 연결하는지 자세히 알 필요는 없으며 Tomcat JDBC, Apache Commons DBCP 그리고 `HikariCP`와 같은 JDBC 커넥션 풀에서 제공하는 DataSource 구현체를 사용하면 됩니다.

#### HikariCP
[`HikariCP`](https://github.com/brettwooldridge/HikariCP)는 가벼움을 자랑하는 JDBC 커넥션 풀 라이브러리입니다. 

```groovy build.gradle
implementation 'com.zaxxer:HikariCP:3.4.5'
implementation 'org.postgresql:postgresql:42.2.16'
implementation 'org.springframework:spring-jdbc:5.2.8.RELEASE'
```

애플리케이션에서 연결할 데이터베이스 정보를 application.properties에 기술합니다.
```properties application.properties
spring.datasource.driver-class-name=org.postgresql.Driver
spring.datasource.url=jdbc:postgresql://localhost:5432/postgres
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.datasource.auto-commit=false
```

HikariCP에서 제공하는 `HikariDataSource`를 DataSource 빈으로 등록합니다.
```java
@PropertySource({"classpath:/application.properties"})
@ComponentScan({
    "com.example.demo.repository",
    "com.example.demo.service"
})
@Configuration
public class DatabaseConfig implements EnvironmentAware {

    private Environment environment;

    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setDriverClassName(environment.getProperty("spring.datasource.driver-class-name", String.class));
        dataSource.setJdbcUrl(environment.getProperty("spring.datasource.url", String.class));
        dataSource.setUsername(environment.getProperty("spring.datasource.username", String.class));
        dataSource.setPassword(environment.getProperty("spring.datasource.password", String.class));
        dataSource.setAutoCommit(environment.getProperty("spring.datasource.auto-commit", Boolean.class, false));
        return dataSource;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
}
```

### JdbcTemplate
[`JdbcTemplate`](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/data-access.html#jdbc-JdbcTemplate)는 SQL을 수행할 수 있는 주요 클래스입니다.

중요한 점은 다음과 같이 다수의 DAO에 대해 공유하기 위하여 안전하게 주입하려면 DataSource에 대한 Setter 주입 시 JdbcTemplate 인스턴스를 생성하는 것이 좋습니다.
```java
@Repository
public class AbstractRepository {

    protected final Logger LOG = LoggerFactory.getLogger(getClass());

    protected DataSource dataSource;
    protected JdbcTemplate jdbcTemplate;

    @Autowired
    private void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
        this.jdbcTemplate = new JdbcTemplate(dataSource);
    }
}
```

이외에도 `SimpleJdbcInsert` 또는 `SimpleJdbcCall`를 사용하여 데이터베이스에 접근할 수 있습니다.

#### SQLExceptionTranslator
SQLExceptionTranslator 인터페이스는 `SQLExceptions`와 `org.springframework.dao.DataAccessException` 사이의 변환을 지원합니다. 기본적으로 `SQLErrorCodeSQLExceptionTranslator` 구현체가 사용됩니다. 

JdbcTemplate.setExceptionTranslator() 함수로 SQLExceptionTranslator를 변경할 수 있습니다.

## Transaction Management
스프링 프레임워크의 `spring-tx` 모듈은 TransactionManager 인터페이스를 통해 트랜잭션 관리를 위한 추상화를 제공합니다.

### TransactionManager
스프링 프레임워크의 TransactionManager 인터페이스는 트랜잭션 전략을 정의합니다. PlatformTransactionManager는 명령형 트랜잭션 관리를 제공합니다. 예를 들어, 다음과 같이 JDBC DataSource에 대한 트랜잭션 지원을 위해 `DataSourceTransactionManager`를 사용할 수 있습니다.
```java
@Bean
public PlatformTransactionManager transactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```

### Declarative transaction management
스프링 프레임워크는 선언적 트랜잭션 관리라는 기능을 제공합니다. 대부분의 스프링 애플리케이션 개발자는 주로 선언적 트랜잭션 관리를 사용합니다. 구성 메타정보 클래스에 `@EnableTransactionManagement`가 선언되면 `@Transactional`를 사용하여 트랜잭션을 관리할 수 있도록 활성화됩니다.

```java
@EnableTransactionManagement
@PropertySource({"classpath:/application.properties"})
@ComponentScan({
    "com.example.demo.repository",
    "com.example.demo.service"
})
@Configuration
public class DatabaseConfig {}
```


### Transaction-bound Events
스프링 애플리케이션 컨텍스트에서 발행되는 이벤트는 `@EventListener`를 선언한 핸들러 함수를 통해 이벤트를 처리할 수 있었습니다. 발행된 이벤트를 처리하는 핸들러 함수에 대한 트랜잭션 관리가 필요하다면 `@TransactionalEventListener`를 대신 사용할 수 있습니다.

다음과 같이 트랜잭션이 진행중인 이벤트인 경우 `@TransactionalEventListener`를 선언하여 처리할 수 있습니다.
```java
@Component
public class UserEventHandler {

    @TransactionalEventListener
    public void handleUserCreatedEvent(CreationEvent<User> creationEvent) {
        // ...
    }
}
```

만약, CreationEvent에 대한 이벤트 발행 시 트랜잭션안에서 처리되었다면 트랜잭션이 완료된 후에 `UserEventHandler`의 `handleUserCreatedEvent()` 핸들러 함수에 의해 이벤트가 처리됩니다.

---

우리는 스프링 프레임워크 기반의 웹 애를리케이션에서 데이터베이스에 접근하기 위한 설정을 완료하였고 비즈니스 로직에서 선언적으로 트랜잭션을 관리하는 방법에 대해 알아보았습니다. 
