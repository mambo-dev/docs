---
title: 스프링 웹 애플리케이션 구성 확장하기
date: 2020-09-12
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

지난 글에서 우리는 스프링 프레임워크 기반의 웹 애플리케이션을 만들고 임베디드 톰캣으로 웹 애플리케이션을 실행해보았습니다. 아직은 웹 애플리케이션에 대해 설정한 것이 없습니다. 그래서 이번에는 [`spring-webmvc`](https://mvnrepository.com/artifact/org.springframework/spring-webmvc/5.2.8.RELEASE)에 포함된 `MVC Configuration`을 통해 웹 애플리케이션에 대한 설정을 확장해나가도록 하겠습니다.

## MVC Configuration
스프링 프레임워크에서 제공하는 MVC Configuration 기능을 활성화하기 위해서 `@EnableWebMvc`를 웹 애플리케이션 컨텍스트 메타 정보 클래스에 선언해야 합니다. @EnableWebMvc가 선언된 경우 애플리케이션 컨텍스트에 `WebMvcConfigurationSupport` 클래스가 자동으로 포함됩니다.

```java
@ComponentScan({"com.example.demo.controller"})
@EnableWebMvc
@Configuration
public class WebConfig {}
```

사실 @EnableWebMvc으로 추가되는 클래스는 WebMvcConfigurationSupport를 확장한 `DelegatingWebMvcConfiguration`입니다. 이 클래스는 스프링 웹 MVC 애플리케이션을 위한 기본 구성을 제공하기도 하며 애플리케이션 컨텍스트에 존재하는 `WebMvcConfigurer` 구현체를 감지하여 설정을 변경할 수 있도록 위임합니다.

### MVC Config API
스프링 웹 MVC에서 제공하는 설정은 `WebMvcConfigurer` 인터페이스를 통해 변경할 수 있습니다.

```java
@ComponentScan({"com.example.demo.controller"})
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {}
```

WebMvcConfigurer 인터페이스에서 제공하는 함수를 통해 변경할 수 있는 기능에 대해서 하나씩 알아보도록 합시다.

### Validation
가장 먼저, [`Bean Validation`](https://beanvalidation.org/)으로 도메인 모델에 대한 검증을 수행할 수 있도록 `Vailidator`에 대한 추상화를 지원합니다. 

도메인 모델 검증을 위해 `hibernate-validator` 의존성을 추가합니다.
```groovy build.gradle
implementation 'org.hibernate.validator:hibernate-validator:6.1.5.Final'
implementation 'org.hibernate.validator:hibernate-validator-annotation-processor:6.1.5.Final'
```

기본적으로 클래스패스에 `Hibernate Validator`와 같은 Bean Validation 의존성이 포함되어있는 경우 전역 Validator로 `LocalValidatorFactoryBean`이 등록됩니다.
```
2020-09-13 10:46:54 [main] INFO  o.h.v.i.util.Version(21) - HV000001: Hibernate Validator 6.1.5.Final
2020-09-13 10:46:54 [main] DEBUG o.h.v.i.e.AbstractConfigurationImpl(203) - Setting custom ConstraintValidatorFactory of type org.springframework.validation.beanvalidation.SpringConstraintValidatorFactory
2020-09-13 10:46:54 [main] DEBUG o.h.v.i.e.AbstractConfigurationImpl(217) - Setting custom ParameterNameProvider of type org.springframework.validation.beanvalidation.LocalValidatorFactoryBean$1
2020-09-13 10:46:54 [main] DEBUG o.h.v.i.x.c.ValidationXmlParser(120) - Trying to load META-INF/validation.xml for XML based Validator configuration.
2020-09-13 10:46:54 [main] DEBUG o.h.v.i.x.c.ResourceLoaderHelper(53) - Trying to load META-INF/validation.xml via user class loader
2020-09-13 10:46:54 [main] DEBUG o.h.v.i.x.c.ResourceLoaderHelper(60) - Trying to load META-INF/validation.xml via TCCL
2020-09-13 10:46:54 [main] DEBUG o.h.v.i.x.c.ResourceLoaderHelper(66) - Trying to load META-INF/validation.xml via Hibernate Validator's class loader
2020-09-13 10:46:54 [main] DEBUG o.h.v.i.x.c.ValidationXmlParser(127) - No META-INF/validation.xml found. Using annotation based configuration only.
```

만약, 전역 Validator를 변경하고 싶은 경우 다음과 같이 `getValidator()`함수를 오버라이딩할 수 있습니다.

```java
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public Validator getValidator() {
        return new LocalValidatorFactoryBean();
    }
}
```

### Interceptors
디스패처 서블릿에 의해 처리가 위임된 웹 요청을 컨트롤러가 담당하기 전에 수행할 작업을 처리하는 것은 인터셉터라는 컴포넌트가 담당합니다. 스프링 웹 MVC는 특정 패턴에 대한 웹 요청에 대해서 인터셉터를 등록할 수 있도록 지원합니다.

인터셉터 등록은 `InterceptorRegistry`를 통해 가능합니다.
```java
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleChangeInterceptor());
    }
}
```

그리고 특정 경로에 대해서만 인터셉터를 적용할 수도 있습니다. 예를 들어, 애플리케이션에서 제공하는 API에 대한 인증을 위해 토큰을 발급하고 검증을 확인하는 작업을 수행할 수 있습니다.
```java
registry.addInterceptor(new HandlerInterceptorAdapter() {}).addPathPatterns("/api/**");
```

> 좀 더 응용하여 웹 요청에 대한 보안을 체크하는 로직이나 처리를 담당하는 컨트롤러의 핸들러 함수가 View를 응답하는 경우에 Model에 공통된 값을 주입하기 위한 작업을 수행할 수 있습니다.

### MessageConverters
스프링 프레임워크는 컨트롤러의 핸들러 함수에 `@ResponseBody`가 선언되어 있다면 반환되는 객체를 메시지로 변환하여 응답하기 위한 작업을 수행합니다. 예를 들어, `Map` 형식을 `JSON` 형식의 메시지로 변경하기 위해서 `ObjectMapper`를 사용하는 `MappingJackson2HttpMessageConverter`를 사용하도록 설정할 수 있습니다.  

우선 아무런 설정 없이 컨트롤러 핸들러 함수에 @ResponseBody를 선언하고 Map 객체를 반환하도록 해보겠습니다.
```java
@GetMapping("/json")
@ResponseBody
public Map<String, Object> json() throws IOException {
    Map<String, Object> body = new HashMap<>();
    body.put("message", "Hello World");
    return body;
}
```

그러면 다음과 같이 기본적으로 Map 형식에 대한 메시지 컨버터가 없으므로 다음과 같은 오류가 발생합니다.
```
org.springframework.http.converter.HttpMessageNotWritableException: No converter found for return value of type: class java.util.HashMap
	org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor.writeWithMessageConverters(AbstractMessageConverterMethodProcessor.java:230)
	org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.handleReturnValue(RequestResponseBodyMethodProcessor.java:181)
	org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite.handleReturnValue(HandlerMethodReturnValueHandlerComposite.java:82)
	org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:123)
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:878)
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:792)
	org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
	org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1040)
	org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:943)
	org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)
	org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898)
	javax.servlet.http.HttpServlet.service(HttpServlet.java:645)
	org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883)
	javax.servlet.http.HttpServlet.service(HttpServlet.java:750)
```

그러면 Map으로 반환하는 것을 JSON 형식의 메시지로 변환하여 응답하도록 메시지 컨버터를 추가해보겠습니다.

[`jackson-databind`](https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind/2.11.2) 의존성을 추가합니다. 
```groovy build.gradle
implementation 'com.fasterxml.jackson.core:jackson-databind:2.11.2'
```

클래스패스에 `jackson-databind` 의존성이 포함되면 자동으로 `MappingJackson2HttpMessageConverter`가 메시지 컨버터로 등록됩니다. 이제 MappingJackson2HttpMessageConverter에 의해 Map 형식이 JSON 형식의 문자열로 변환되어 응답됩니다.

```json
{"message":"Hello World"}
```

그리고 `configureMessageConverters()` 함수를 통해 메시지 컨버터를 등록할 수 있습니다. 예를 들어, MappingJackson2HttpMessageConverter에서 사용하는 `ObjectMapper`를 변경할 수 도 있습니다.
```java
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder objectMapperBuilder = new Jackson2ObjectMapperBuilder()
                .indentOutput(true);
        converters.add(new MappingJackson2HttpMessageConverter(objectMapperBuilder.build()));
    }
}
```

위 예제 코드에서 ObjectMapper의 `indentOutput` 속성을 활성화하였으므로 다음과 같이 응답 메시지가 출력됩니다.
```json
{
  "message" : "Hello World"
}
```

### View Resolvers
지난 글에서 프리마커 템플릿 엔진을 사용하여 HTML을 응답하기 위해 FreeMarkerViewResolver를 빈 클래스로 등록하였습니다. MVC Configuration에서 제공하는 `configureViewResolvers()`함수를 통해 좀더 쉽게 FreeMarkerViewResolver를 빈으로 등록할 수 있습니다.

```java
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.freeMarker()
                .cache(false)
                .suffix(".html");
    }
}
```

클래스패스에 [`jackson-databind`](https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind/2.11.2)이 포함되면 `ContentNegotiatingViewResolver`가 JSON 형식의 메시지를 처리할 수 있도록 `MappingJackson2JsonView`를 등록합니다.

ContentNegotiatingViewResolver는 웹 요청 경로에 확장자가 표현되는 경우 해당 형식으로 응답할 수 있도록 지원합니다. 예를 들어, `.json`으로 끝나는 요청에 대하여 MappingJackson2JsonView으로 응답하는 것을 말합니다.

```java
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.enableContentNegotiation(new MappingJackson2JsonView());
    }
}
```

### Static Resources
때때로, 웹 애플리케이션에서는 자바스크립트, 스타일시트 또는 이미지 파일과 같은 `정적 리소스`를 제공해야할 수 있습니다. 그리고 이 정적 리소스는 사용자 또는 응답한 HTML에서 요구할 가능성이 있습니다.

스프링 MVC Configuration는 정적 리소스를 처리할 수 있도록 `Resource Handler`를 지원합니다.

리소스 핸들러 등록은 `addResourceHandlers()`함수에서 `ResourceHandlerRegistry`를 통해 가능합니다.
```java
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/images/**", "/css/**", "/js/**")
                .addResourceLocations("classpath:/static/images/", "classpath:/static/css/", "classpath:/static/js/")
                .setCacheControl(CacheControl.maxAge(Duration.ofDays(1)));
    }
}
```

위와 같이 설정된 경우 `/images`, `/css`, `/js`로 시작되는 요청은 리소스 핸들러가 담당하도록 처리되며 리소스 핸들러가 찾아야할 리소스의 위치는 클래스패스에 있는 `static` 폴더입니다.

다음과 같이 http://localhost:8080/images/snorlax.jpg 요청에 대해 `static/images/snorlax.jpg` 경로의 파일을 찾아 응답하게 됩니다.

![](/images/posts/spring5-002.PNG)

## Advanced MVC Configuration
스프링 프레임워크 기반의 웹 애플리케이션을 위해 도움이 되는 몇가지 설정에 대해 알아보겠습니다.

### Filters
`spring-web` 모듈은 웹 애플리케이션에서 활용할 수 있는 유용한 필터 클래스들을 제공합니다.

#### FormContentFilter
브라우저는 기본적으로 HTTP.GET 또는 HTTP.POST을 통해 폼 데이터를 전송합니다. 브라우저가 아닌 클라이언트들은 PUT, PATCH, DELETE를 HTTP 메소드로도 사용할 수 있습니다. 그런데 Servlet API는 `ServletRequest.getParameter*()`가 HTTP POST에 대해서만 접근할 수 있도록 허용합니다.

`spring-web` 모듈에서 제공하는 `FormContentFilter`는 `application/x-www-form-urlencoded`를 컨텐트 타입으로 요청하는 PUT, PATH, DELETE에 대해서 요청 바디를 폼 데이터로 읽을 수 있도록 `ServletRequest`에 대해 랩핑을 지원합니다.

```java
@Bean
public FormContentFilter formContentFilter() {
    FormContentFilter formContentFilter = new FormContentFilter();
    formContentFilter.setCharset(StandardCharsets.UTF_8);
    formContentFilter.setFormConverter(new FormHttpMessageConverter());
    return formContentFilter;
}
```

#### ForwardedHeaderFilter
로드 밸런서와 같이 프록시를 통해 들어오는 요청은 호스트, 포트 그리고 스키마가 변경될 가능성이 있습니다. 이에 스프링 웹 모듈에서 제공하는 `ForwardedHeaderFilter`는 RFC 7239로 정의된 `Forwared HTTP Header`에 대해서 본래의 요청 정보를 제공합니다.

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
@Bean
public ForwardedHeaderFilter forwardedHeaderFilter() {
    return new ForwardedHeaderFilter();
}
```

### CORS
스프링 웹 MVC는 [`CORS(Cross-Origin Resource Sharing)`](https://developer.mozilla.org/ko/docs/Web/HTTP/CORS)를 처리할 수 있도록 지원합니다. 

```java
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedMethods(HttpMethod.GET.name(), HttpMethod.POST.name(), HttpMethod.PUT.name(), HttpMethod.DELETE.name())
                .allowCredentials(true)
                .maxAge(Duration.ofHours(1).toSeconds());
    }
}
```

### HTTP Caching
HTTP 캐시는 웹 애플리케이션 성능을 향상시킬 수 있는 기능입니다. HTTP 캐시는 `Cache-Control` 응답 헤더와 함께 `Last-Modified`와 `ETag`와 같은 요청 헤더에 의해 이루어집니다. 스프링 프레임워크에서는 CacheControl를 통해 캐시 정책을 나타낼수 있습니다.

```java
CacheControl cacheControl = CacheControl.maxAge(1, TimeUnit.DAYS).noTransform().cachePublic();
```

스프링 컨트롤러는 `ResponseEntity`를 통해 `CacheControl`을 적용할 수 있도록 지원합니다.

```java
@RestController
@RequestMapping("/api")
public class ApiController {
    @GetMapping("/health")
    public ResponseEntity<Object> health() {
        return ResponseEntity
                .ok()
                .cacheControl(CacheControl
                                .maxAge(1, TimeUnit.MINUTES)
                                .cachePublic()
                                .noTransform())
                .body(true);
    }
}
```

> 눈치채지 못하셨을수도 있겠지만 `리소스 핸들러`를 등록할 때에도 캐시 정책을 지정하였습니다.

### Web Security
스프링 프레임워크는 보안 관련 기능을 `Spring Security` 프로젝트를 통해 제공합니다. 웹 보안과 관련된 부분은 스프링 시큐리티 모듈에 대한 글을 통해 알아보도록 합니다.

---

스프링 프레임워크 기반의 웹 애플리케이션을 위한 기본적인 설정이나 기능에 대해서 추가해보았습니다. 앞으로는 웹 애플리케이션의 목표인 다음과 같은 것들을 적용하는 방법에 대해 다루도록 하겠습니다.

- 스프링 테스트 모듈을 통한 애플리케이션 기능 테스트
- 웹 관련 기능 확장
- 구글 SMTP 서버를 활용한 이메일 발송
- 반복적인 작업을 수행하는 스케줄링 적용
- 스프링 JDBC를 활용한 데이터베이스 액세스
- 스프링 HTTP 요청 클라이언트 활용
- 스프링 세션을 활용한 사용자 세션 관리 적용
- 스프링 시큐리티를 활용한 웹 보안 적용
