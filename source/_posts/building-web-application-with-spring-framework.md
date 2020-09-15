---
title: 스프링 웹 애플리케이션 만들기
date: 2020-09-11
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

지난 글에서는 스프링 프레임워크의 기본이 되는 IoC 컨테이너에 대하여 알아보고 애플리케이션 컨텍스트라는 IoC 컨테이너를 직접 구성해보았습니다. 이번 글에서는 스프링 프레임워크의 [`spring-webmvc`](https://mvnrepository.com/artifact/org.springframework/spring-webmvc/5.2.8.RELEASE) 의존성을 추가하여 웹 애플리케이션을 만들어보도록 하겠습니다.

## Web on Servlet Stack
스프링 5부터는 웹 애플리케이션이 동작하는 방식이 기존의 서블릿 API를 사용하는 서블릿 스택과 [Reactive Streams](https://www.reactive-streams.org/) 스펙 기반의 [리액티브](https://projectreactor.io/) 스택이 있습니다. 리액티브 스택은 블로킹 기반의 서블릿 스택과 달리 논-블로킹이라는 특징이 있습니다. 리액티브 스택의 웹 애플리케이션을 개발하기 위해서는 우선적으로 비동기 프로그래밍을 배워야하므로 많은 개발자들에게 익숙한 서블릿 스택 기반의 웹 애플리케이션을 만들어보도록 합니다.

서블릿 스택 기반의 웹 애플리케이션을 만들기 위해 `spring-webmvc`와 `javax.servlet-api` 의존성을 추가합니다.
```groovy
implementation 'org.springframework:spring-webmvc:5.2.8.RELEASE'
implementation 'javax.servlet:javax.servlet-api:4.0.1'
```

### WebApplicationInitializer
Tomcat과 같은 웹 컨테이너에서 웹 애플리케이션을 실행할 때 [`web.xml`](https://cloud.google.com/appengine/docs/flexible/java/configuring-the-web-xml-deployment-descriptor?hl=ko)이라는 배포 설명자 파일을 읽습니다. `spring-web` 모듈에서 제공하는 [`WebApplicationInitializer`](https://docs.spring.io/spring/docs/5.2.8.RELEASE/javadoc-api/org/springframework/web/WebApplicationInitializer.html) 인터페이스는 `web.xml`을 대체할 수 있도록 지원합니다.

다시 말해, 컨테이너는 클래스패스 위치에 web.xml이 없을 경우 WebApplicationInitializer를 web.xml으로 대체하여 읽습니다.

이전 글에서 SpringApplication에서 구성하였던 애플리케이션 컨텍스트를 WebApplicationInitializer의 onStartup(ServletContext servletContext) 메소드에서 구성하고 ContextLoaderListener와 DispatherServlet을 기술합니다.

```java
public class WebServletInitializer implementation WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext servletContext) {
        AnnotationConfigWebApplicationContext applicationContext = new AnnotationConfigWebApplicationContext();
        applicationContext.setConfigLocation("com.example.demo.config");
        servletContext.addListener(new ContextLoaderListener(applicationContext));
        ServletRegistration.Dynamic dispatcher = servletContext.addServlet("dispatcher", new DispatcherServlet(applicationContext));
        dispatcher.setLoadOnStartup(1);
        dispatcher.addMapping("/");
    }
}
```

위 자바 코드는 아래의 web.xml을 기술한 것과 같습니다.
```xml web.xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://java.sun.com/xml/ns/javaee  http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">
       <context-param>
              <param-name>contextConfigLocation</param-name>
              <param-value>/WEB-INF/spring/application-context.xml</param-value>
       </context-param>
       
       <listener>
              <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
       </listener>
       
       <servlet>
              <servlet-name>dispatcher</servlet-name>
              <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
              <init-param>
                     <param-name>contextConfigLocation</param-name>
                     <param-value>/WEB-INF/spring/application-context.xml</param-value>
              </init-param>
              <load-on-startup>1</load-on-startup>
       </servlet>
              
       <servlet-mapping>
              <servlet-name>dispatcher</servlet-name>
              <url-pattern>/</url-pattern>
       </servlet-mapping>
</web-app>
```

web.xml에 대한 예제를 찾아보면 루트 컨텍스트와 서블릿 컨텍스트를 나누어서 구성하는 것이 많습니다. 스프링 프레임워크는 WebApplicationInitializer를 확장한 `AbstractAnnotationConfigDispatcherServletInitializer`를 통해서 루트 애플리케이션 컨텍스트와 서블릿 애플리케이션 컨텍스트로 나누어지는 컨텍스트 계층을 구성할 수 있도록 지원합니다.

![](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/images/mvc-context-hierarchy.png)

위 그림은 스프링 프레임워크 공식 레퍼런스에서 제공하는 다중 컨텍스트 계층을 표현합니다. 루트 애플리케이션 컨텍스트는 웹 레이어와 상관없는 비즈니스 로직의 빈들을 관리하며 서블릿 애플리케이션 컨텍스트는 웹과 관련된 빈들이 있다고 설명합니다.

다음 예제 코드는 AbstractAnnotationConfigDispatcherServletInitializer를 적용하는 예시입니다.
```java
public class WebServletInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { AppConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { WebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}
```

getRootConfigClasses에서 반환하는 클래스는 루트 애플리케이션 컨텍스트에서 컨테이너를 구성하기 위해 참조하며 getServletConfigClasses에서 반환하는 클래스는 서블릿 애플리케이션 컨텍스트를 구성하기 위해 참조합니다.

그리고 getServletMappings는 디스패처 서블릿을 생성할 때 매핑되는 경로를 나타냅니다. 이때, 디스패처 서블릿 이름은 dispatcher을 기본으로 사용합니다. 만약, 디스패처 서블릿 이름을 변경하고 싶다면 getServletName() 메소드를 오버라이딩하세요.

```java
public abstract class AbstractDispatcherServletInitializer extends AbstractContextLoaderInitializer {
	public static final String DEFAULT_SERVLET_NAME = "dispatcher";
}
```

이제 앞으로 추가될 웹과 관련된 구성 메타정보 클래스는 WebConfig가 담당하며 공통으로 사용하거나 비즈니스 로직에서 사용하는 빈들은 AppConfig에서 관리하도록 하겠습니다. 

톰캣에서 읽어야할 배포 설명자인 WebApplicationInitializer가 준비되었으므로 웹 애플리케이션을 구동할 수 있습니다. 만약, 스프링 부트 프로젝트를 기반으로 애플리케이션 개발을 시작한다면 내장된 서버를 사용하여 구동할 수 있습니다. 내장된 서버 중 하나인 임베디드 톰캣 모듈을 활용하여 톰캣을 설치하지 않아도 웹 애플리케이션을 구동할 수 있습니다.

### Embedded Tomcat
톰캣을 실행하기 위하여 `tomcat-embed-core`와 `tomcat-embed-jasper` 의존성을 추가합니다.
```groovy build.gradle
implementation 'org.apache.tomcat.embed:tomcat-embed-core:9.0.37'
implementation 'org.apache.tomcat.embed:tomcat-embed-jasper:9.0.37'
```

SpringApplication에서 톰캣 서버를 실행하도록 다음과 같이 코드를 작성합니다. 
```java
public class SpringApplication {
    public static void main(final String[] args) throws LifecycleException {
        Tomcat tomcat = new Tomcat();
        Connector connector = tomcat.getConnector();
        connector.setURIEncoding(StandardCharsets.UTF_8.displayName());

        tomcat.addWebapp("", new File("src/main/webapp").getAbsolutePath());
        tomcat.setPort(8080);
        tomcat.start();
        tomcat.getServer().await();
    }
}
```

저는 `srg/main/webapp` 폴더를 생성하여 톰캣이 해당 경로를 webapp으로 사용하도록 하였습니다. 이제 SpringApplication를 실행하면 톰캣이 구동되면서 다음과 같이 로그가 출력됩니다.

```java
...
9월 11, 2020 10:39:10 오후 org.apache.catalina.startup.ContextConfig getDefaultWebXmlFragment
정보: No global web.xml found
9월 11, 2020 10:39:11 오후 org.apache.catalina.core.ApplicationContext log
정보: 1 Spring WebApplicationInitializers detected on classpath
...
9월 11, 2020 10:39:12 오후 org.apache.coyote.AbstractProtocol start
정보: Starting ProtocolHandler ["http-nio-8080"]
```

출력된 로그를 살펴보면 Spring WebApplicationInitializers detected on classpath 에서 확인할 수 있듯이 클래스패스에 있는 WebApplicationInitializer를 web.xml으로 사용하였습니다.

이제 웹 애플리케이션의 메인 경로인 http://localhost:8080 를 브라우저를 통해 요청해봅니다.

```java
경고: No mapping for GET /
```

디스패처 서블릿이 요청을 처리할 서블릿을 찾을 수 없었고 결국 응답할 수 없음을 나타내었습니다. 이제 메인경로에 대한 요청을 담당할 빈 클래스를 만들어보도록 하겠습니다.

### Annotated Controllers
스프링 웹 MVC에서 웹 요청을 처리할 컴포넌트는 `@Controller`와 `@RestController`를 선언하여 등록할 수 있습니다. 그리고 이 컴포넌트들을 서블릿 컨텍스트 구성 메타정보 클래스인 WebConfig으로 등록되도록 합니다.

```java
@ComponentScan({"com.example.demo.controller"})
@Configuration
public class WebConfig {}
```

이제 com.example.demo.controller 패키지에 만들어지는 컨트롤러 컴포넌트들은 서블릿 컨텍스트에 등록됩니다. 다음과 같이 HomeController라는 컨트롤러를 만들고 메인 경로를 처리할 함수를 작성해보겠습니다.

```java
@Controller
public class HomeController {
    @GetMapping("/")
    public void home(HttpServletRequest request, HttpServletResponse response) throws IOException {
        PrintWriter writer = response.getWriter();
        writer.println("Hello World");
        writer.flush();
    }
}
```

이제 SpringApplication을 실행하면 디스패처 서블릿이 웹 요청에 대해 처리할 컨트롤러를 찾아 위임하여 HTTP 요청을 처리하고 응답을 받게됩니다. `Hello World`라는 문자열이 표시되었나요?

대부분의 스프링 애플리케이션 예제와 달리 요청을 처리하는 함수가 void를 반환하는지 궁금할 수 있습니다. 이제 컨트롤러를 어떻게 작성해야하는지 자세히 알아보도록 하겠습니다. 

#### Request Mapping
가장 먼저 [`@RequestMapping`](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/web.html#mvc-ann-requestmapping)은 선언된 함수가 어떻게 웹 요청을 처리할지 설정할 수 있습니다. 예를 들어, 특정 URL, 파라미터, 헤더, 미디어 타입에 따라 처리할 요청을 구분할 수 있습니다. 위 예제에서 선언된 `@GetMapping`은 이 @RequestMapping에 대해 HTTP 메소드에 따라 확장한 어노테이션 중 하나입니다.

만약, 컨트롤러에서 처리하는 함수가 여러가지 HTTP 메소드를 지원하는 경우가 아니라면 HTTP 메소드에 따라 확장된 어노테이션을 선언하는 것이 가독성에 이점이 있습니다.

#### Handler Methods
컨트롤러에 선언된 핸들러 함수는 여러가지 [매개변수](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/web.html#mvc-ann-arguments)를 받을 수 있습니다. 위 예제 코드에서 HttpServletRequest와 HttpServletResponse도 매개변수로 받을 수 있는 클래스 중 하나입니다. 

#### Return Values
위 예제 코드에서 home() 핸들러 함수가 void 형식을 반환한 이유는 컨트롤러에 선언된 핸들러 함수가 반환할 수 있는 유형이 정해져있기 때문입니다.

예를 들어, 컨트롤러에 선언된 핸들러 함수가 [반환하는 유형](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/web.html#mvc-ann-return-types) 중 `String`은 문자열을 응답하는 것이 아닌 뷰 이름을 지정하는 것으로 정해져있습니다. 

따라서, "Hello World"라는 문자열을 응답으로 출력 위하여 String 형식으로 반환하였다면 디스패처 서블릿을 응답을 위해 Hello Wolrd라는 이름을 가진 뷰(응답 객체)를 찾게됩니다. 결국 요청을 처리할 수 없다고 판단하고 응답을 받지 못하게 됩니다.

다음 처럼 말이죠.
```java
@GetMapping("/")
public String home(HttpServletRequest request, HttpServletResponse response) throws IOException {
    return "Hello World";
}

경고: No mapping for GET /Hello World
```

스프링 웹 MVC는 핸들러 함수에서 반환한 유형에 따라 올바른 응답을 하기 위해서 `ViewResolver` 인터페이스를 통해 응답 객체로 변환됩니다. 기본적으로 ViewResolver에 대한 설정이 없으면 InternalResourceViewResolver를 사용합니다.

여기서 우리는 컨트롤러를 통해 요청을 처리하고 컨트롤러의 핸들러 함수가 반환하는 형식에 따라 응답 유형이 다를 수 있다는 것을 알았습니다.

## Logging System
스프링 프레임워크 기반의 웹 애플리케이션은 실행하였지만 어떻게 동작하고 있는지 궁금하고 출력되는 로그가 정해져있어 답답하기도 합니다. 스프링 프레임워크는 [`spring-jcl`](https://mvnrepository.com/artifact/org.springframework/spring-jcl) 모듈을 통해 `SLF4J`와 같은 다양한 로깅에 대한 추상화를 지원합니다. 

따라서, `slf4j-api` 의존성을 추가하여 스프링 프레임워크에 대한 로그 출력을 활성화할 수 있습니다. 
```groovy
implementation 'org.slf4j:slf4j-api:1.7.30'
// implementation 'org.slf4j:slf4j-simple:1.7.30'
```

SLF4J는 로거 추상화로 구현체를 아직 추가하지 않았기 때문에 다음과 같은 로그가 출력됩니다.
```java
9월 12, 2020 4:21:31 오후 org.apache.jasper.servlet.TldScanner scanJars
정보: At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
```

가장 기본적인 `slf4j-simple` 의존성을 추가하면 정상적으로 스프링 프레임워크에 대한 로그가 출력됩니다.

### Logback
[`Logback`](http://logback.qos.ch/)은 여러가지 SLF4J 구현체 중 하나로 Log4j를 확장한 프로젝트로 Logback을 추가하여 스프링 프레임워크에 대한 로그를 출력하겠습니다.

다음 `logback-classic` 의존성을 추가합니다.
```groovy
implementation 'ch.qos.logback:logback-classic:1.2.3'
```

클래스패스에 `logback.xml` 파일을 생성하고 다음처럼 로그백에 대한 설정을 기술합니다.
```java logback.xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %highlight(%-5level) %cyan(%logger{25}\(%line\)) - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="debug">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

이제 애플리케이션 실행 시 스프링 프레임워크에 대한 로그가 출력됩니다. 
```sh
2020-09-12 16:58:11 [main] INFO  o.s.w.s.DispatcherServlet(525) - Initializing Servlet 'dispatcher'
2020-09-12 16:58:11 [main] DEBUG o.s.w.c.s.AnnotationConfigWebApplicationContext(596) - Refreshing WebApplicationContext for namespace 'dispatcher-servlet'
2020-09-12 16:58:11 [main] DEBUG o.s.w.c.s.AnnotationConfigWebApplicationContext(217) - Registering component classes: [class com.example.demo.config.WebConfig]
2020-09-12 16:58:11 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory(217) - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalConfigurationAnnotationProcessor'
2020-09-12 16:58:11 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory(217) - Creating shared instance of singleton bean 'org.springframework.context.event.internalEventListenerProcessor'
2020-09-12 16:58:11 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory(217) - Creating shared instance of singleton bean 'org.springframework.context.event.internalEventListenerFactory'
2020-09-12 16:58:11 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory(217) - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalAutowiredAnnotationProcessor'
2020-09-12 16:58:11 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory(217) - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalCommonAnnotationProcessor'
2020-09-12 16:58:11 [main] DEBUG o.s.u.c.s.UiApplicationContextUtils(85) - Unable to locate ThemeSource with name 'themeSource': using default [org.springframework.ui.context.support.DelegatingThemeSource@7741507c]
2020-09-12 16:58:11 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory(217) - Creating shared instance of singleton bean 'webConfig'
2020-09-12 16:58:11 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory(217) - Creating shared instance of singleton bean 'homeController'
2020-09-12 16:58:11 [main] DEBUG o.s.w.s.m.m.a.RequestMappingHandlerMapping(351) - 1 mappings in 'org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping'
2020-09-12 16:58:11 [main] DEBUG o.s.w.s.m.m.a.RequestMappingHandlerAdapter(611) - ControllerAdvice beans: none
2020-09-12 16:58:11 [main] DEBUG o.s.w.s.m.m.a.ExceptionHandlerExceptionResolver(294) - ControllerAdvice beans: none
2020-09-12 16:58:11 [main] DEBUG o.s.w.s.DispatcherServlet(542) - enableLoggingRequestDetails='false': request parameters and headers will be masked to prevent unsafe logging of potentially sensitive data
2020-09-12 16:58:11 [main] INFO  o.s.w.s.DispatcherServlet(547) - Completed initialization in 165 ms
```

이제 우리는 애플리케이션에서 인터페이스를 통해 로그를 출력할 수 있게 되었습니다. 이제 애플리케이션 개발자가 할일은 요구사항에 따라 logback.xml을 작성하는 일입니다.

## Template Engine
웹 애플리케이션의 목표는 웹 요청을 처리하여 응답하는 것입니다. 그리고 HTML 형식으로 응답하는 것은 중요합니다. 스프링 웹 MVC는 기본적으로 [`InternalResourceViewResolver`](https://docs.spring.io/spring-framework/docs/5.2.8.RELEASE/javadoc-api/org/springframework/web/servlet/view/InternalResourceViewResolver.html)을 사용합니다. InternalResourceViewResolver는 UrlBasedViewResolver를 확장한 클래스로 Servlet이나 JSP와 같은 `InternalResourceView` 또는 `JstlView`를 지원하게 됩니다.

예를 들어, 다음과 같이 컨트롤러 핸들러 함수에서 index라는 뷰 이름을 반환합니다.
```java
@Controller
public class HomeController {
    @GetMapping("/")
    public String index(HttpServletRequest request, HttpServletResponse response) throws IOException {
        return "index";
    }
}
```

그러면 InternalResourceViewResolver에 의해 InternalResourceView로 판단하고 `/index`라는 경로로 요청이 `포워딩` 됩니다.
```sh
2020-09-13 08:19:33 [http-nio-8080-exec-1] DEBUG o.s.w.s.DispatcherServlet(91) - GET "/", parameters={}
2020-09-13 08:19:33 [http-nio-8080-exec-1] DEBUG o.s.w.s.m.m.a.RequestMappingHandlerMapping(414) - Mapped to com.example.demo.controller.HomeController#index(HttpServletRequest, HttpServletResponse)
2020-09-13 08:19:33 [http-nio-8080-exec-1] DEBUG o.s.w.s.v.InternalResourceView(309) - View name 'index', model {}
2020-09-13 08:19:33 [http-nio-8080-exec-1] DEBUG o.s.w.s.v.InternalResourceView(169) - Forwarding to [index]
2020-09-13 08:19:33 [http-nio-8080-exec-1] DEBUG o.s.w.s.DispatcherServlet(91) - "FORWARD" dispatch for GET "/index", parameters={}
2020-09-13 08:19:33 [http-nio-8080-exec-1] WARN  o.s.w.s.PageNotFound(1251) - No mapping for GET /index
2020-09-13 08:19:33 [http-nio-8080-exec-1] DEBUG o.s.w.s.DispatcherServlet(1127) - Exiting from "FORWARD" dispatch, status 404
2020-09-13 08:19:33 [http-nio-8080-exec-1] DEBUG o.s.w.s.DispatcherServlet(1131) - Completed 404 NOT_FOUND
```

결과적으로 우리는 클래스패스에 생성한 index.html을 응답할 수 없습니다.

### FreeMarkerViewResolver
대부분의 개발자가 JSP와 같은 뷰 기술에 익숙할 수 있지만 스프링 프레임워크에서는 JSP를 선호하지 않습니다. 그래서 스프링 부트 프로젝트에서는 기본적으로 JSP를 뷰로 지원하지 않습니다. [`FreeMarkerViewResolver`](https://docs.spring.io/spring/docs/5.2.8.RELEASE/javadoc-api/org/springframework/web/servlet/view/freemarker/FreeMarkerViewResolver.html)는 `FreeMarkerView` 클래스를 뷰로 응답할 수 있도록 지원합니다. 따라서, `FreeMarker Template Engine`을 사용하여 index.html을 뷰로 응답할 수 있습니다.

먼저, 프리마커 템플릿 엔진을 사용하기 위해 [`freemarker`](https://mvnrepository.com/artifact/org.freemarker/freemarker/2.3.30)와 `spring-context-support` 의존성을 추가합니다.
```groovy
implementation 'org.freemarker:freemarker:2.3.30'
implementation 'org.springframework:spring-context-support:5.2.8.RELEASE'
```



서블릿 컨텍스트 메타정보 클래스에 `FreeMarkerViewResolver`와 `FreeMarkerConfigurer`를 빈으로 등록합니다.
```java
@ComponentScan({"com.example.demo.controller"})
@Configuration
public class WebConfig {

    @Bean
    public FreeMarkerViewResolver freemarkerViewResolver() {
        FreeMarkerViewResolver viewResolver = new FreeMarkerViewResolver();
        viewResolver.setContentType("text/html; charset=UTF-8");
        viewResolver.setCache(false);
        viewResolver.setSuffix(".html");
        return viewResolver;
    }

    @Bean
    public FreeMarkerConfigurer freemarkerConfig() {
        FreeMarkerConfigurer freeMarkerConfigurer = new FreeMarkerConfigurer();
        freeMarkerConfigurer.setTemplateLoaderPath("classpath:/templates/");
        return freeMarkerConfigurer;
    }
}
```

이제 애플리케이션을 실행하면 FreeMarkerViewResolver와 FreeMarkerConfigurer를 빈으로 등록하는 로그가 출력되었습니다.
```
2020-09-13 09:36:59 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory(217) - Creating shared instance of singleton bean 'freemarkerViewResolver'
2020-09-13 09:36:59 [main] DEBUG o.s.b.f.s.DefaultListableBeanFactory(217) - Creating shared instance of singleton bean 'freemarkerConfig'
2020-09-13 09:36:59 [main] DEBUG o.s.w.s.v.f.FreeMarkerConfigurer(347) - Template loader path [class path resource [templates/]] resolved to file path [C:\Users\Mambo\git\spring5\out\production\resources\templates]
```

이제 http://localhost:8080에 대해 웹 요청이 들어오면 `/templates/index.html`을 `FreeMarkerView`로 응답하게 됩니다.

```
2020-09-13 09:38:10 [http-nio-8080-exec-2] DEBUG o.s.w.s.DispatcherServlet(91) - GET "/", parameters={}
2020-09-13 09:38:10 [http-nio-8080-exec-2] DEBUG o.s.w.s.m.m.a.RequestMappingHandlerMapping(414) - Mapped to com.example.demo.controller.HomeController#index(HttpServletRequest, HttpServletResponse)
2020-09-13 09:38:10 [http-nio-8080-exec-2] DEBUG o.s.w.s.v.f.FreeMarkerView(309) - View name 'index', model {}
2020-09-13 09:38:10 [http-nio-8080-exec-2] DEBUG o.s.w.s.v.f.FreeMarkerView(176) - Rendering [index.html]
2020-09-13 09:38:10 [http-nio-8080-exec-2] DEBUG o.s.w.s.DispatcherServlet(1131) - Completed 200 OK
```

#### Model
FreeMarker도 JSP처럼 Model에 등록된 속성을 표시할 수 있습니다.

```java
@GetMapping("/")
public String index(HttpServletRequest request, HttpServletResponse response, Model model) throws IOException {
    model.addAttribute("message", "Hello World");
    return "index";
}
```

이제 다음과 같이 Model에 message 속성을 index.html 템플릿에서 사용할 수 있습니다.

```html
<html lang="ko">
<head>
    <title></title>
</head>
<body>
<h3>${message}</h3>
</body>
</html>
```

이제 할일은 [FreeMarker Template](https://freemarker.apache.org/docs/dgui_template_overallstructure.html) 방식에 따라 템플릿을 구성하면 됩니다.

---

이번 글을 통해 스프링 프레임워크 기반의 웹 애플리케이션을 만들고 실행하는 법에 대해서 알게 되었습니다. 스프링 MVC 모듈은 [`MVC Configuration`](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/web.html#mvc-config)을 위한 클래스를 제공합니다. 다음 글에서는 MVC Configuration을 통해 스프링 웹 애플리케이션에 좀 더 확장된 기능을 적용해보도록 하겠습니다.