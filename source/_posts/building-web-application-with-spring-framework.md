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

지난 글에서 우리는 스프링 프레임워크의 기본이 되는 개념인 IoC 컨테이너라는 애플리케이션 컨텍스트에 대해 배웠습니다. 이번 글에서는 스프링 프레임워크 기반의 웹 애플리케이션을 구성하고 실행하는 방법에 대해서 다루겠습니다.

## Web on Servlet Stack
스프링 5 부터 웹 애플리케이션이 동작하는 방식이 기존의 서블릿 API를 사용하는 서블릿 스택과 [Reactive Streams](https://www.reactive-streams.org/) 스펙 기반의 [리액티브](https://projectreactor.io/) 스택이 있습니다. 리액티브 스택은 블로킹 기반의 서블릿 스택과 달리 논-블로킹이라는 특징이 있습니다. 리액티브 스택의 웹 애플리케이션을 개발하기 위해서는 우선적으로 비동기 프로그래밍을 배워야하므로 많은 개발자들에게 익숙한 서블릿 스택 기반의 웹 애플리케이션을 만들어보도록 합니다.

서블릿 스택 기반의 웹 애플리케이션을 만들기 위해 [`spring-webmvc`](https://mvnrepository.com/artifact/org.springframework/spring-webmvc/5.2.8.RELEASE)와 `javax.servlet-api` 의존성을 추가합니다.
```groovy
implementation 'org.springframework:spring-webmvc:5.2.8.RELEASE'
implementation 'javax.servlet:javax.servlet-api:4.0.1'
```

자바 웹 애플리케이션을 Tomcat과 같은 웹 컨테이너를 통해 실행할 경우 웹 컨테이너는 클래스패스에 위치한 [`web.xml`](https://cloud.google.com/appengine/docs/flexible/java/configuring-the-web-xml-deployment-descriptor?hl=ko)이라는 배포 설명자 파일을 참조합니다. 스프링 프레임워크의 `spring-web` 모듈에는 배포 설명자 파일인 web.xml을 자바 코드로 대체할 수 있는 특별한 인터페이스를 제공합니다.

### WebApplicationInitializer
웹 컨테이너에서 가장 먼저 참조하는 배포 설명자를 구성하기 위하여 [`WebApplicationInitializer`](https://docs.spring.io/spring/docs/5.2.8.RELEASE/javadoc-api/org/springframework/web/WebApplicationInitializer.html) 인터페이스에 대한 구현체를 생성할 수 있습니다.

웹 컨테이너가 실행될 때 클래스패스에 위치한 web.xml 파일이 존재하지 않을 경우 WebApplicationInitializer를 감지하여 web.xml로 대체하여 사용하게 됩니다. 정말로 WebApplicationInitializer로 web.xml을 대체할 수 있는지 확인해봅시다.

우선 다음과 같이 WebApplicationInitializer 구현체를 클래스패스에 생성합니다.
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

위 자바 코드는 아래의 web.xml을 기술한 것과 같다고 볼 수 있습니다.
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

위 예제는 `단일 애플리케이션 컨텍스트`로 동작하는 웹 애플리케이션을 구성합니다. 그러나 대부분의 web.xml에 대한 예제를 찾아보면 `루트 애플리케이션 컨텍스트`와 디스패처 서블릿이 참조하는 `서블릿 컨텍스트`를 구분하여 구성한 것이 많습니다. 스프링 프레임워크는 단일 애플리케이션 컨텍스트 구조 뿐만 아니라 루트 애플리케이션 컨텍스트와 서블릿 애플리케이션 컨텍스트를 나누어 컨텍스트 계층을 구성할 수 있는 특별한 `AbstractAnnotationConfigDispatcherServletInitializer` 클래스를 제공합니다.

AbstractAnnotationConfigDispatcherServletInitializer가 클래스패스에 위치하는 경우 다음과 같은 구조로 웹 애플리케이션을 구성할 수 있습니다.
![](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/images/mvc-context-hierarchy.png)

위 그림은 스프링 프레임워크 공식 레퍼런스에서 제공하는 `다중 컨텍스트 계층`을 표현합니다. 웹과 관련된 빈 클래스들은 서블릿 웹 애플리케이션 컨텍스트에서 관리되며 루트 애플리케이션 컨텍스트에는 웹과 관련이 없거나 공통적으로 사용되는 빈 클래스들을 관리하도록 합니다.

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

루트 애플리케이션 컨텍스트가 참조하는 구성 메타정보는 `getRootConfigClasses`에서 반환합니다. 그리고 서블릿 애플리케이션 컨텍스트에서 참조하는 구성 메타정보는 `getServletConfigClasses`에서 반환합니다. 따라서, 위 예제 코드에서는 루트 애플리케이션 컨텍스트는 AppConfig 클래스에 등록된 빈 클래스들을 관리하게 되고 WebConfig 클래스에 등록된 빈 클래스들은 서블릿 애플리케이션 컨텍스트를 통해 관리됩니다. 

> 이때 만들어지는 디스패처 서블릿은 dispatcher라는 이름을 기본으로 사용합니다.

따라서, 다중 애플리케이션 컨텍스트 계층으로 웹 애플리케이션을 구성한다면 루트 애플리케이션 컨텍스트와 서블릿 애플리케이션 컨텍스트에서 관리해야할 빈들을 잘 구분하여 등록하여 사용하는 것이 좋습니다.

이제 웹 컨테이너에서 읽어야할 배포 설명자인 WebApplicationInitializer가 준비되었으므로 웹 애플리케이션을 구동할 수 있습니다. 저는 임베디드 톰캣 모듈을 통하여 쉽게 애플리케이션을 구동하는 방법을 설명하도록 하겠습니다.

### Embedded Tomcat
임베디드 톰캣으로 애플리케이션을 실행하기 위하여 `tomcat-embed-core`와 `tomcat-embed-jasper` 의존성을 추가합니다.
```groovy build.gradle
implementation 'org.apache.tomcat.embed:tomcat-embed-core:9.0.37'
implementation 'org.apache.tomcat.embed:tomcat-embed-jasper:9.0.37'
```

SpringApplication에서 톰캣 서버를 실행하도록 다음과 같이 코드를 작성합니다. 
```java
public class SpringApplication {
    public static void main(final String[] args) throws LifecycleException {
        Tomcat tomcat = new Tomcat();
        tomcat.setBaseDir("out/webapp");
        Connector connector = tomcat.getConnector();
        connector.setURIEncoding(StandardCharsets.UTF_8.displayName());

        tomcat.addWebapp("", new File("src/main/webapp").getAbsolutePath());
        tomcat.setPort(8080);
        tomcat.start();
        tomcat.getServer().await();
    }
}
```

이제 SpringApplication을 실행하고 로그를 확인합니다.

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

톰캣이 웹 애플리케이션을 정상적으로 구동하였으므로 브라우저를 통해 http://localhost:8080 로 접속해보겠습니다.

```java
경고: No mapping for GET /
```

이런... 디스패처 서블릿이 처리를 위임할 서블릿을 찾을 수 없어 경고 메시지가 출력되었고 브라우저에서는 응답 없음을 확인할 수 있습니다. 이제 우리가 알아야할 것은 웹 요청을 처리할 서블릿을 스프링 프레임워크가 제공하는 클래스로 구성하는 것입니다.

### Annotated Controllers
스프링 웹 MVC 모듈은 웹 요청을 처리할 컴포넌트를 만들 수 있는 어노테이션을 제공합니다. 스프링 프레임워크에서 웹 요청을 처리하는 컴포넌트를 컨트롤러라고 합니다. 그리고 컨트롤러라는 것을 `@Controller`와 `@RestController`를 선언하여 나타낼 수 있습니다. 

컨트롤러는 웹 관련 빈 클래스이므로 서블릿 웹 애플리케이션 컨텍스트의 구성 메타정보인 `WebConfig`에 등록하겠습니다.
```java
@ComponentScan({"com.example.demo.controller"})
@Configuration
public class WebConfig {}
```

`@ComponentScan`을 선언하였으므로 `com.example.demo.controller` 패키지에 있는 컨트롤러 컴포넌트들은 서블릿 애플리케이션 컨텍스트에서 관리됩니다. 

다음과 같이 `HomeController`라는 컨트롤러를 만들어 웹 요청을 처리할 `핸들러 함수`를 만들겠습니다.
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

이제 SpringApplication을 실행하면 디스패처 서블릿이 웹 요청에 대해 처리할 컨트롤러를 찾아 위임하여 HTTP 요청을 처리하고 응답을 받게됩니다. 

`Hello World`라는 문자열이 브라우저에 표시되었나요?

대부분의 스프링 애플리케이션 예제와 달리 컨트롤러의 핸들러 함수에서 "Hello World"를 반환하지 않는지 궁금해 해야합니다. 저는 어떤 지식을 공부할 때에는 궁금한게 많아야한다고 생각하는 편입니다. 왜 HttpServletResponse에서 `PrintWriter`를 가져와 문자열을 출력했는지는 공식 레퍼런스를 살펴보면 찾을 수 있습니다.

우선 컨트롤러 컴포넌트를 만들때 사용하는 것들에 대해 알아보면서 설명하겠습니다.

#### Request Mapping
가장 먼저 [`@RequestMapping`](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/web.html#mvc-ann-requestmapping)은 선언된 핸들러 함수가 어떻게 웹 요청을 처리할지 결정할 수 있습니다. 예를 들어, 특정 URL, 파라미터, 헤더, 미디어 타입에 따라 처리할 요청을 구분할 수 있습니다. 위 예제에서 선언된 `@GetMapping`은 이 @RequestMapping에 대해 HTTP 메소드에 따라 확장한 어노테이션 중 하나입니다.

- @GetMapping
- @PostMapping
- @PutMapping
- @DeleteMapping

만약, 컨트롤러에서 처리하는 함수가 여러가지 HTTP 메소드를 지원하는 경우가 아니라면 HTTP 메소드에 따라 확장된 어노테이션을 선언하는 것이 가독성에 이점이 있습니다.

#### Handler Methods
컨트롤러에 선언된 핸들러 함수는 여러가지 [매개변수](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/web.html#mvc-ann-arguments)를 지정할 수 있습니다.

위 예제 코드에서 HttpServletRequest와 HttpServletResponse도 매개변수로 받을 수 있는 클래스 중 하나입니다. 

#### Return Values
앞서 핸들러 함수에서 void 형식을 반환한 이유를 여기서 확인할 수 있습니다. 컨트롤러의 핸들러 함수는 반환할 수 있는 형식이 정해져 있습니다. 예를 들어, 컨트롤러에 선언된 핸들러 함수가 [반환하는 유형](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/web.html#mvc-ann-return-types) 중 `String`은 문자열을 응답하는 것이 아니라 스프링 프레임워크에서 사용하는 `뷰(View)`라는 응답 객체의 이름을 지정하는 것으로 정해져있습니다. 

따라서, "Hello World"라는 문자열을 응답으로 출력 위하여 String 형식으로 반환하였다면 디스패처 서블릿은 응답을 위해 Hello Wolrd라는 이름을 가진 뷰를 찾게됩니다. 디스패처 서블릿은 결국 요청을 처리할 수 없다고 판단합니다.

다음 처럼 말이죠.
```java
@GetMapping("/")
public String home(HttpServletRequest request, HttpServletResponse response) throws IOException {
    return "Hello World";
}

경고: No mapping for GET /Hello World
```

스프링 웹 MVC는 핸들러 함수에서 반환하는 유형에 따라 응답해야하는 것을 구분하기 위하여 `ViewResolver` 인터페이스를 통해 수행합니다. 기본적으로 ViewResolver에 대한 설정이 존재하지 않으면 `InternalResourceViewResolver`를 사용하게 됩니다. 여기서 확인할 수 있는 점은 우리가 원하는 형식으로 응답하기 위해서는 ViewResolver에 대한 설정을 해야한다는 것입니다.

다음의 ViewResolver에 대해서 찾아보시기를 추천합니다.

- [ContentNegotiatingViewResolver](https://docs.spring.io/spring-framework/docs/5.2.8.RELEASE/javadoc-api/org/springframework/web/servlet/view/ContentNegotiatingViewResolver.html)
- [InternalResourceViewResolver](https://docs.spring.io/spring-framework/docs/5.2.8.RELEASE/javadoc-api/org/springframework/web/servlet/view/InternalResourceViewResolver.html)
- [FreeMarkerViewResolver](https://docs.spring.io/spring-framework/docs/5.2.8.RELEASE/javadoc-api/org/springframework/web/servlet/view/freemarker/FreeMarkerViewResolver.html)

## Template Engine
위에서 언급한 ViewResolver 중 `FreeMarkerViewResolver`는 프리마커 템플릿 엔진을 사용하여 HTML을 응답하기 위한 설정을 할 때 사용할 수 있습니다. 웹 애플리케이션의 목표는 웹 요청을 처리하여 응답하는 것이며 HTML 형식으로 응답하는 것은 중요한 부분입니다.

스프링 웹 MVC에서 기본적으로 [`InternalResourceViewResolver`](https://docs.spring.io/spring-framework/docs/5.2.8.RELEASE/javadoc-api/org/springframework/web/servlet/view/InternalResourceViewResolver.html)을 사용하여 뷰를 응답한다고 하였습니다. 이 InternalResourceViewResolver는 UrlBasedViewResolver를 확장한 클래스로 Servlet이나 JSP와 같은 `InternalResourceView` 또는 `JstlView`를 응답으로 사용하게 됩니다.

예를 들어, 다음과 같이 컨트롤러 핸들러 함수에서 `index`라는 뷰 이름을 반환합니다.
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

> 여기서 포워딩 되는 이유는 InternalResourceViewResolver가 UrlBasedViewResolver를 기반으로 하기 때문입니다.

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

결과적으로 우리는 클래스패스에 HTML 파일을 생성하였다고 해서 응답할 수 없습니다.

### FreeMarkerViewResolver
스프링 프레임워크 애플리케이션을 개발하는 대부분의 개발자들은 JSP를 사용해왔습니다. 그러나 최근에는 JSP 보다는 Thymeleaf 또는 FreeMarker 같은 템플릿 엔진을 선호합니다. 그래서 스프링 부트 프로젝트에서는 기본적으로 JSP에 대한 의존성을 지원하지 않고 있습니다. JSP를 원하는 개발자들이 서운해할 수 있지만 프리마커 템플릿 엔진을 사용하여 HTML을 응답해보도록 하겠습니다.

[`FreeMarkerViewResolver`](https://docs.spring.io/spring/docs/5.2.8.RELEASE/javadoc-api/org/springframework/web/servlet/view/freemarker/FreeMarkerViewResolver.html)는 `FreeMarkerView` 클래스를 뷰로 응답할 수 있도록 지원합니다. 즉, 프리마커 템플릿 엔진을 사용하여 웹 요청을 HTML 형식으로 응답할 수 있게 되는 것입니다.

먼저, 프리마커 템플릿 엔진을 사용하기 위해 [`freemarker`](https://mvnrepository.com/artifact/org.freemarker/freemarker/2.3.30)와 `spring-context-support` 의존성을 추가합니다.

```groovy build.gradle
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
FreeMarker도 템플릿 엔진이므로 Model에 값을 넣어 표시할 수 있습니다.

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

## Logging System
스프링 프레임워크는 [`spring-jcl`](https://mvnrepository.com/artifact/org.springframework/spring-jcl) 모듈을 통해 `SLF4J`와 같은 다양한 로깅에 대한 추상화를 지원합니다. 따라서, 스프링 프레임워크 기반의 웹 애플리케이션에서 로그 출력을 위한 기능은 쉽게 적용할 수 있습니다.

`slf4j-api` 의존성을 추가합니다.
```groovy build.gradle
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
slf4j-simple로 출력되는 로그 보다는 [`Logback`](http://logback.qos.ch/)과 같은 SLF4J 구현체를 사용하는 것이 좋습니다.

다음 `logback-classic` 의존성을 추가합니다.
```groovy build.gradle
implementation 'ch.qos.logback:logback-classic:1.2.3'
```

클래스패스에 `logback.xml` 파일을 생성하고 로그백으로 로그를 출력하기 위한 설정을 기술합니다.
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

이제 애플리케이션을 실행하면 웹 애플리케이션에 대한 로그가 다음과 같이 출력됩니다.
```
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

---

이번 글을 통해 스프링 프레임워크 기반의 웹 애플리케이션을 만들고 실행하는 법에 대해서 알게 되었습니다. 스프링 MVC 모듈은 [`MVC Configuration`](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/web.html#mvc-config)을 위한 클래스를 추가적으로 제공합니다. 다음 글에서는 MVC Configuration을 통해 스프링 프레임워크 기반의 웹 애플리케이션에 대한 기능을 좀 더 확장해보겠습니다.