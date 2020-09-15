---
title: 스프링 웹 애플리케이션 구성 확장하기
date: 2020-09-12
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
description: 잠만보와 함께하는 스프링 프레임워크 기반의 웹 애플리케이션
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

지난 [`스프링 웹 애플리케이션 만들기`](../building-web-application-with-spring-framework)를 통해 스프링 웹 애플리케이션을 만들고 임베디드 톰캣을 통해 애플리케이션을 실행해보았습니다. 이번에는 [`spring-webmvc`](https://mvnrepository.com/artifact/org.springframework/spring-webmvc/5.2.8.RELEASE)에 포함된 `MVC Configuration`을 통해 웹 애플리케이션에 대한 기능을 확장해보도록 하겠습니다.

## MVC Configuration
스프링이 제공하는 MVC Configuration을 활성화하기 위해서는 `@EnableWebMvc`를 웹 애플리케이션 컨텍스트 메타 정보 클래스에 선언하면 됩니다. @EnableWebMvc가 선언되면 `WebMvcConfigurationSupport`가 애플리케이션 컨텍스트에 자동으로 포함됩니다.

```java
@ComponentScan({"com.example.demo.controller"})
@EnableWebMvc
@Configuration
public class WebConfig {}
```

사실은 @EnableWebMvc 선언 시 추가되는 클래스는 `DelegatingWebMvcConfiguration` 입니다. 이 클래스는 스프링 MVC 애플리케이션을 위한 기본 구성을 제공하며 `WebMvcConfigurer` 구현체를 감지하며 설정을 변경할 수 있도록 위임합니다.

### MVC Config API
`WebMvcConfigurer` 인터페이스를 통해 MVC 설정을 변경할 수 있는 함수를 제공합니다. 

```java
@ComponentScan({"com.example.demo.controller"})
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {}
```

WebMvcConfigurer 인터페이스에서 제공하는 함수를 통해 변경할 수 있는 기능에 대해서 알아보도록 합시다.


### Validation
기본적으로 클래스패스에 `Hibernate Validator`와 같은 Bean Validation 의존성이 포함되어있는 경우 `LocalValidatorFactoryBean`이 전역 Vailidator로 등록됩니다. 

```groovy
implementation 'org.hibernate.validator:hibernate-validator:6.1.5.Final'
implementation 'org.hibernate.validator:hibernate-validator-annotation-processor:6.1.5.Final'
```

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

만약, 전역 Validator를 변경하고 싶은 경우 다음과 같이 getValidator() 함수를 오버라이딩할 수 있습니다.

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
스프링 웹 MVC는 특정 패턴에 대한 웹 요청에 대해 컨트롤러가 처리를 담당하기 전에 수행할 작업을 처리할 수 있도록 인터셉터를 등록할 수 있도록 지원합니다. 

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

예를 들어, 로케일을 변경하거나 웹 요청에 대해 보안을 체크하는 것 또는 처리를 담당할 컨트롤러의 핸들러 함수가 View를 반환하는 경우 Model에 값을 주입하는 것을 수행할 수 있습니다.

### MessageConverters
스프링 웹 MVC에서 기본적으로 JSON 형식의 메시지에 대한 컨버터에서 ObjectMapper를 사용합니다. configureMessageConverters() 함수를 통해 메시지 컨버터를 등록할 수 있도록 지원합니다.

컨트롤러 핸들러 함수에서 Map을 반환하고 `@ResponseBody`을 선언하여 반환되는 객체를 응답 바디로 설정합니다.
```java
@GetMapping("/json")
@ResponseBody
public Map<String, Object> json() throws IOException {
    Map<String, Object> body = new HashMap<>();
    body.put("message", "Hello World");
    return body;
}
```

그러면 핸들러 함수에 @ResponseBody가 선언되어있는 경우 응답 형식에 대한 컨버터를 찾습니다. 기본적으로는 Map 형식에 대한 메시지 컨버터가 없으므로 다음과 같은 오류가 발생합니다.
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

Map 형식을 JSON 형식의 문자열로 변환하기 위하여 [`jackson-databind`](https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind/2.11.2) 의존성을 추가합니다. 
```groovy
implementation 'com.fasterxml.jackson.core:jackson-databind:2.11.2'
```

`jackson-databind` 의존성이 클래스패스에 포함되는 경우 자동으로 `MappingJackson2HttpMessageConverter`가 메시지 컨버터로 등록됩니다. 이제 MappingJackson2HttpMessageConverter에 의해 Map 형식이 JSON 형식의 문자열로 변환되어 응답됩니다.

```json
{"message":"Hello World"}
```

MappingJackson2HttpMessageConverter에서 사용하는 ObjectMapper를 변경하기 위하여 configureMessageConverters를 활용할 수 있습니다.
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

이제 응답된 JSON 문자열에 Intent가 적용되었음을 확인할 수 있습니다.
```json
{
  "message" : "Hello World"
}
```

### View Resolvers
지난 글에서 FreeMarkerViewResolver를 직접 빈으로 등록하였습니다. Spring MVC는 configureViewResolvers() 함수를 통해 좀 더 쉽게 FreeMarkerViewResolver를 등록할 수 있도록 지원합니다.

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

Spring MVC는 [`jackson-databind`](https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind/2.11.2) 의존성이 클래스패스에 포함될 경우 MappingJackson2JsonView를 ContentNegotiatingViewResolver에서 지원할 수 있도록 활성화합니다. configureViewResolvers() 함수에서도 `ContentNegotiatingViewResolver`에 대해 설정할 수 있습니다.
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
스프링 MVC Configuration은 JS, CSS 또는 이미지 파일과 같은 `정적 리소스`를 편리하게 배포할 수 있도록 `리소스 핸들러`를 등록하는 것을 지원합니다.

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

/images, /css, /js로 시작되는 요청은 리소스 핸들러가 담당하도록 설정하며 리소스 핸들러가 담당할 리소스의 위치는 클래스패스에 `static` 폴더를 지정하였습니다.

다음과 같이 http://localhost:8080/images/snorlax.jpg 요청에 대해 static/images/snorlax.jpg라는 파일을 찾아 응답하게 됩니다.

![](/images/posts/spring5-002.PNG)

## Advanced MVC Configuration
MVC Configuration를 확장하는 기능에 대해 알아보도록 하겠습니다.

### Filters
`spring-web` 모듈은 유용한 필터 클래스들을 제공합니다.

#### FormContentFilter
브라우저는 기본적으로 HTTP.GET 또는 HTTP.POST을 통해 폼 데이터를 전송합니다. 브라우저가 아닌 클라이언트들은 PUT, PATCH, DELETE를 HTTP 메소드로도 사용할 수 있습니다. 

Servlet API는 `ServletRequest.getParameter*()`가 HTTP POST에 대해서만 접근할 수 있도록 허용합니다.

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
로드 밸런서와 같이 프록시를 통해 들어오는 요청은 호스트, 포트 그리고 스키마가 변경될 수 있습니다. `ForwardedHeaderFilter`는 RFC 7239로 정의된 `Forwared HTTP Header`에 대해서 본래의 요청 정보를 제공합니다.

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
@Bean
public ForwardedHeaderFilter forwardedHeaderFilter() {
    return new ForwardedHeaderFilter();
}
```

### CORS
스프링 MVC는 `CORS(Cross-Origin Resource Sharing)`를 처리할 수 있도록 지원합니다. 

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
HTTP 캐시는 웹 애플리케이션 성능을 향상시킬 수 있는 기능입니다. HTTP 캐시는 `Cache-Control` 응답 헤더와 함께 `Last-Modified`와 `ETag`와 같은 요청 헤더에 의해 이루어집니다. 

```java
CacheControl cacheControl = CacheControl.maxAge(1, TimeUnit.DAYS).noTransform().cachePublic();
```

스프링 컨트롤러는 ResponseEntity를 통해 CacheControl을 적용할 수 있도록 지원합니다.

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

### Web Security
스프링 프레임워크는 보안 관련 기능을 `Spring Security` 프로젝트를 통해 제공합니다. 웹 보안과 관련된 부분은 스프링 시큐리티 모듈에 대한 글을 통해 알아보도록 합니다.

---

이제부터는 애플리케이션이 요구하는 기능에 따라 구성을 확장하는 예제를 구분하여 작성합니다. 하나의 글에서 다루기엔 내용이 많아질 수 있으므로 원하는 기능을 어떻게 적용하는지 참고하시기 바랍니다.
