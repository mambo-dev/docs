---
title: Spring Framework 시작하기
date: 2020-09-08
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

## Getting Started
[start.spring.io](https://start.spring.io/)를 통해 스프링 부트 기반의 애플리케이션을 쉽게 만들 수 있습니다! 스프링 부트가 무엇이냐구요? 신경쓰지마세요. 그냥 스프링 프레임워크를 좀 더 쉽게 적용할 수 있는 [프로젝트](https://spring.io/projects)입니다. 즉, 스프링 부트 기반의 애플리케이션은 스프링 프레임워크로 작성된 애플리케이션 입니다.

다음과 같이 그레이들 프로젝트와 자바 11 기반으로 애플리케이션을 구성할 경우 다음과 같이 구성됨을 확인할 수 있습니다.

![](/images/posts/getting-started-with-spring-boot-01.PNG)

사실 스프링 프레임워크는 구성하는게 전부라고 할 정도로 구성하는 부분도 많고 시간이 많이 걸립니다. 그래서 스프링 애플리케이션을 개발할 때 기본적으로 구성하는 부분들을 별다른 설정없이도 적용할 수 있게 해주는 것이 스프링 부트 프로젝트입니다.

스프링 프레임워크 기반으로 애플리케이션을 개발하다보면 사용자의 세션 정보를 관리하기 위해서 Spring Session 프로젝트를 추가한다거나 애플리케이션에 보안을 적용하기 위해서 Spring Security 프로젝트를 추가하는 것과 같은 이치라고 보면 됩니다.

스프링 프로젝트들이 무조건 필요하다는 것은 아니지만 애플리케이션을 개발하기 전에 요구사항을 적용할 수 있는 프로젝트를 먼저 찾아보는 것도 나쁘지 않습니다. 직접 개발하는 것보다 쉽게 적용할 수도 있고 소요 시간이 줄어드는 효과도 있습니다.

## IoC Container
스프링 프레임워크에서 빈 클래스들은 자신이 사용하고 있는 다른 빈 클래스들을 생성자 또는 팩토리 메소드의 매개변수 그리고 인스턴스가 생성된 이후에 프로퍼티로써 설정됩니다. 이렇게 빈 클래스가 만들어질때 의존성을 넣어주는 것을 컨테이너가 담당합니다. 

`org.springframework.beans`와 `org.springframework.context` 패키지에는 스프링 프레임워크 IoC 컨테이너의 기초가 되며 `BeanFactory`라는 인터페이스는 모든 유형의 객체를 관리할 수 있는 구성 방식을 제공합니다.

[`ApplicationContext`](https://docs.spring.io/spring-framework/docs/5.2.8.RELEASE/javadoc-api/org/springframework/context/ApplicationContext.html)는 빈 팩토리의 서브 인터페이스로써 AOP, 메시지 처리 또는 이벤트 퍼블리싱같은 추가 기능을 제공합니다. 따라서, BeanFactory를 사용하는 것보다 ApplicationContext를 주로 사용하게 됩니다.

또한, 다음과 같이 특정 애플리케이션 레이어에서 사용하기 위한 애플리케이션 컨텍스트 서브 인터페이스가 있습니다. 예를 들어, WebApplicationContext 인터페이스 같은 경우 `getServletContext()`라는 함수를 추가적으로 제공하여 현재 웹 애플리케이션의 서블릿 컨텍스트를 가져올 수 있습니다.

![](https://docs.spring.io/spring/docs/current/spring-framework-reference/images/container-magic.png)

스프링 프레임워크에서 애플리케이션 컨텍스트는 위와 같이 동작합니다. 애플리케이션에서 요구하는 구성 메타정보와 함께 빈 클래스가 되는 객체를 제공하면 애플리케이션 컨텍스트를 통해 언제든지 등록된 정보를 가져올 수 있습니다.

스프링 프레임워크는 원래 구성 메타정보를 간단하고 직관적인 XML로 기술하였습니다. 그러나 요즘 대부분의 개발자들은 스프링 3.0 부터 제공하는 자바 기반의 구성 메타정보를 기술합니다. 

### Java-based Container Configuration
간단하게 그레이들 프로젝트를 생성하여 자바 기반으로 구성 메타정보를 기술해보도록 하겠습니다. 먼저, 인텔리제이를 통해 그레이들 프로젝트를 시작합니다.

그리고 다음과 같이 build.gradle에 [`spring-context`](https://mvnrepository.com/artifact/org.springframework/spring-context/5.2.8.RELEASE) 의존성을 추가합니다.
```groovy build.gradle
plugins {
    id 'java'
}

group 'org.example'
version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework:spring-context:5.2.8.RELEASE'
    testCompile group: 'junit', name: 'junit', version: '4.12'
}
```

그리고 다음 그림처럼 SpringApplication을 생성합니다.
![](/images/posts/spring5-001.PNG)

```java
@Configuration
public class AppConfig {}

public class BasicService {}
    
public class SpringApplication {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        BasicService basicService = applicationContext.getBean(BasicService.class);
        System.out.println(basicService);
    }
}
```

그리고 SpringApplication을 실행해보면 ApplicationContext으로 AppConfig라는 구성 메타정보를 주입하였으나 구성 메타정보에는 아무런 정보도 기술하지 않았으므로 다음과 같이 오류가 발생합니다. 

```java
Connected to the target VM, address: '127.0.0.1:14173', transport: 'socket'
Exception in thread "main" org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'service.BasicService' available
    at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBean(DefaultListableBeanFactory.java:352)
    at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBean(DefaultListableBeanFactory.java:343)
    at org.springframework.context.support.AbstractApplicationContext.getBean(AbstractApplicationContext.java:1127)
    at SpringApplication.main(SpringApplication.java:8)
Disconnected from the target VM, address: '127.0.0.1:14173', transport: 'socket'
```

AppConfig 구성 메타정보에 BasicService 유형의 빈을 등록합니다. 빈을 등록할 때에는 `@Configuration`을 선언한 구성 메타 정보 클래스에서 `@Bean`을 선언합니다.
```java
@Configuration
public class AppConfig {
    @Bean
    public BasicService basicService() {
        return new BasicService();
    }
}
```

SpringApplication을 다시 실행해봅시다. 
```java
Connected to the target VM, address: '127.0.0.1:14529', transport: 'socket'
service.BasicService@23c30a20
Disconnected from the target VM, address: '127.0.0.1:14529', transport: 'socket'
```

ApplicationContext에서 BasicService라는 유형의 빈을 요청하였고 ApplicationContext는 AppConfig에 의해 등록된 BasicService 빈을 제공하였습니다. 

### @CompnentScan
@Configuration를 선언한 클래스에 @CompnentScan을 선언하여 패키지 단위로 빈을 등록하게 할 수 있습니다. 우선 AppConfig에 @CompnentScan을 선언하여 service 패키지를 지정합니다.

```java
@ComponentScan(basePackages = {"service"})
@Configuration
public class AppConfig {}
```

그리고 service 패키지에 ConversionService라는 클래스를 만들고 @Component를 선언합니다. SpringApplication에서 BasicService가 아닌 ConversionService 유형의 빈을 요청하도록 수정한 후 실행합니다.

```java
@Component
public class ConversionService {}
    
public class SpringApplication {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        ConversionService conversionService = applicationContext.getBean(ConversionService.class);
        System.out.println(conversionService);
    }
}
```

다음과 같이 정상적으로 ConversionService 빈이 존재함을 확인하였습니다.
```java
Connected to the target VM, address: '127.0.0.1:14978', transport: 'socket'
service.ConversionService@1c5920df
Disconnected from the target VM, address: '127.0.0.1:14978', transport: 'socket'
```

간단하게 그레이들 프로젝트를 생성하여 스프링 의존성을 추가하고 자바 기반으로 컨테이너를 구성하였습니다. 결과적으로 다음과 같이 정리할 수 있습니다.

- 구성 메타정보는 @Configuration을 선언하여 만든다.
- @Configuration을 선언한 구성 메타정보 클래스에서 @Bean을 선언하여 빈을 등록할 수 있다.
- @CompnentScan을 선언하여 패키지 단위로 빈을 등록할 수도 있다.

## Additional Capabilities
애플리케이션 컨텍스트에서 추가적으로 제공하는 기능 중 자주 활용되는 것에 대해 알아보도록 하겠습니다. 애플리케이션에서 사용되는 프로퍼티 속성을 관리하거나 국제화 메시지 지원을 위한 메시지 소스, 파일과 같은 리소스에 대한 접근, 그리고 이벤트 발행이 있습니다.

### Environment Abstraction
애플리케이션 컨텍스트는 Environment 인터페이스를 통하여 애플리케이션 환경별로 프로파일을 지정하거나 애플리케이션에서 사용되는 환경변수와 같은 프로퍼티를 관리합니다.

#### @Profile
@Configuration을 선언한 구성 메타정보 클래스에 [`@Profile`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-definition-profiles-java)을 선언하여 애플리케이션 환경을 구분할 수 있습니다. @Profile이 선언된 구성 메타정보에 기술된 빈은 해당 프로파일이 활성화되어있을때만 등록됩니다.

앞서 만들었던 AppConfig 클래스에 @Profile을 선언하고 실행합니다.

```java
@Profile({"mambo"})
@ComponentScan(basePackages = {"service"})
@Configuration
public class AppConfig {}
```

프로파일을 지정하면 다음과 같이 애플리케이션 컨텍스트로부터 EventService 빈을 가져올 수 없습니다.
```java
Exception in thread "main" org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'service.EventService' available
    at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBean(DefaultListableBeanFactory.java:352)
    at org.springframework.beans.factory.support.DefaultListableBeanFactory.getBean(DefaultListableBeanFactory.java:343)
    at org.springframework.context.support.AbstractApplicationContext.getBean(AbstractApplicationContext.java:1127)
    at SpringApplication.main(SpringApplication.java:9)
```

애플리케이션 컨텍스트가 참조하는 Environment에 프로파일을 지정하고 실행하도록 코드를 약간 수정합니다.

```java
public class SpringApplication {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        environment.setActiveProfiles("mambo");
        applicationContext.register(AppConfig.class);
        applicationContext.refresh();
        EventService eventService = applicationContext.getBean(EventService.class);
        System.out.println(eventService);
    }
}

```

mambo라는 프로파일을 지정하였기에 다음과 같이 EventService 빈을 가져올 수 있게 됩니다.
```java
service.EventService@4ef74c3
```

#### @PropertySource
@Configuration을 선언한 구성 메타정보 클래스에 [`@PropertySource`](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-using-propertysource)을 선언하여 `.properties`파일에 기술된 프로퍼티 정보를 쉽게 Environment에 주입할 수 있습니다.

클래스패스에 .properties 파일을 생성하여 프로퍼티를 추가합니다.
```properties application.properties
spring.application.name=SpringApplication
```

생성한 프로퍼티 파일을 @PropertySource를 선언하여 지정하고 Environment를 통해 프로퍼티를 가져와봅니다.
```java
@PropertySource({"classpath:application.properties"})
@Profile({"mambo"})
@ComponentScan(basePackages = {"service"})
@Configuration
public class AppConfig {}
    
public class SpringApplication {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        environment.setActiveProfiles("mambo");
        applicationContext.register(AppConfig.class);
        applicationContext.refresh();
        String property = environment.getProperty("spring.application.name");
        System.out.println("property: "+ property);
    }
}
```

다음과 같이 프로퍼티 파일에 추가한 프로퍼티가 출력됩니다.
```java
property: SpringApplication
```

### Internationalization
애플리케이션 컨텍스트는 MessageSource 인터페이스를 통하여 `i18n`이라고 하는 국제화 기능을 지원합니다. 애플리케이션 컨텍스트가 로드되었을때 컨텍스트에 선언된 `MessageSource`라는 이름의 빈을 자동으로 찾아 메시지 소스로 사용합니다.

주로 ResourceBundleMessageSource라는 MessageSource 구현체를 사용하게 되며 이 구현체의 대안으로 핫 리로드가 되는 ReloadableResourceBundleMessageSource도 있습니다.

먼저, 메시지 소스로 사용할 .properties파일을 클래스패스에 추가합니다.
```properties messages.properties
argument.required=The {0} argument is required.
```

구성 메타정보 클래스인 AppConfig에 ResourceBundleMessageSource를 빈으로 등록합니다.
```java
@PropertySource({"classpath:application.properties"})
@Profile({"mambo"})
@ComponentScan(basePackages = {"service"})
@Configuration
public class AppConfig {
    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        messageSource.setBasenames("messages");
        return messageSource;
    }
}
```

그리고 애플리케이션 컨텍스트에서 메시지 소스에 등록된 메시지를 가져올 수 있습니다.

```java
public class SpringApplication {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        environment.setActiveProfiles("mambo");
        applicationContext.register(AppConfig.class);
        applicationContext.refresh();
        String message = applicationContext.getMessage("argument.required", new Object[]{"username"}, Locale.getDefault());
        System.out.println(message);
    }
}
```

다음과 같이 메시지의 {0}부분이 매개변수 "username"로 치환하여 제공하였습니다.
```java
The username argument is required.
```

### Access Resources 
애플리케이션 컨텍스트는 ResourceLoader 인터페이스를 통하여 `Resource` 오브젝트를 불러올 수 있습니다. Resource는 클래스패스, 파일시스템, 표준 URL등 다양한 방식으로 로우-레벨의 리소스를 가져올 수 있습니다.

클래스패스에 README.md 파일을 만들고 애플리케이션 컨텍스트로부터 파일을 가져와봅니다.

```java
public class SpringApplication {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        environment.setActiveProfiles("mambo");
        applicationContext.register(AppConfig.class);
        applicationContext.refresh();
        
        try {
            Resource resource = applicationContext.getResource("classpath:README.md");
            String readme = StreamUtils.copyToString(resource.getInputStream(), StandardCharsets.UTF_8);
            System.out.println(readme);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

다음과 같이 README.md 파일 내용이 출력되었습니다. 
```java
An application context is a ResourceLoader, which can be used to load Resource objects.
```

또한, ApplicationContext의 ResourceLoader가 아니더라도 스프링에서 제공하는 ClasspathResource를 통해 클래스패스에 위치한 파일을 가져올 수 있습니다.
```java
ClassPathResource resource = new ClassPathResource("README.md");
String readme = StreamUtils.copyToString(resource.getInputStream(), StandardCharsets.UTF_8);
System.out.println(readme);

An application context is a ResourceLoader, which can be used to load Resource objects.
```

### Event Publication
애플리케이션 컨텍스트는 `ApplicationEventPublisher` 인터페이스를 통하여 이벤트를 발행하고 `EventListener`를 선언한 메소드를 통해 발행된 이벤트를 처리할 수 있습니다.

```java
public class SpringApplication {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        environment.setActiveProfiles("mambo");
        applicationContext.register(AppConfig.class);
        applicationContext.refresh();
        applicationContext.publishEvent("Published!");
    }
}
```

이벤트 발행은 끝났으나 발행된 이벤트를 수신하는 이벤트 리스너를 등록해야합니다. ApplicationListener를 구현하거나 @EventListener를 선언하여 메소드가 이벤트를 수신할 수 있게 합니다.

```java
@Component
public class EventService {
    @EventListener
    public void handle(String message) {
        System.out.println(message);
        // Published!
    }
}
```

간단하게 문자열 유형의 이벤트를 수신하였으나 POJO에 대한 특정 이벤트를 발행하여 수신할 수 있습니다. 예를 들어, 애플리케이션 컨텍스트의 `start()` 메소드를 호출하면 ContextStartedEvent 이벤트가 발행됩니다.
```java
applicationContext.start();

@EventListener
public void startUp(ContextStartedEvent startedEvent) {
    System.out.println("Context started : " + startedEvent.getSource());
    // Context started : org.springframework.context.annotation.AnnotationConfigApplicationContext@51cdd8a, started on Thu Sep 10 23:48:00 KST 2020
}
```

---

스프링 프레임워크 문서에서는 [Vailidation](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#validation), [Data Binding](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-beans-conventions), [SpEL](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#expressions), AOP에 대해서 추가적으로 기술하고 있습니다. 이 부분에 대해서는 웹 애플리케이션과 관련하여 주로 다루게되므로 넘어가도록 하겠습니다.

스프링 프레임워크 기반의 애플리케이션이 반드시 웹 애플리케이션이 되는 것은 아닙니다. 하지만 스프링 프레임워크는 WebApplicationContext를 제공하여 웹 애플리케이션을 위한 애플리케이션 컨텍스트를 구성할 수 있도록 지원합니다. 

다음 글에서는 [스프링 프레임워크 기반의 웹 애플리케이션](../building-web-application-with-spring-framework)을 만들어보는 시간을 가지겠습니다. 