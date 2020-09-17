---
title: 스프링 웹 애플리케이션 - HTTP 요청 클라이언트
date: 2020-09-15
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

스프링 프레임워크 기반의 웹 애플리케이션에서 다른 웹 애플리케이션으로 요청해서 데이터를 받아와야하는 요구사항이 있을 수 있습니다. 이번 글에서는 스프링 프레임워크에서 제공하는 HTTP 요청 기능에 대해 알아보도록 하겠습니다.

## Rest Clients  
스프링 웹 애플리케이션에서 다른 웹 애플리케이션으로 HTTP 요청을 수행할 수 있는 HTTP 요청 클라이언트 클래스를 제공합니다.

### RestTemplate
[`RestTemplate`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)는 서블릿 스택 기반의 웹 애플리케이션에서 사용할 수 있는 `동기형 HTTP 요청 클라이언트` 클래스입니다.

동기 방식의 `RestTemplate`는 다음과 같이 사용할 수 있습니다.
```java
private static final String SERVICE_KEY = "";
@Test
public void TEST_001() throws ParserConfigurationException, IOException, SAXException {
    RestTemplate restTemplate = new RestTemplate();

    UriComponents uriComponents = UriComponentsBuilder
            .fromHttpUrl("http://openapi.data.go.kr/openapi/service/rest/Covid19/getCovid19InfStateJson")
            .queryParam("serviceKey", SERVICE_KEY)
            .queryParam("pageNo", 1)
            .queryParam("numOfRows", 10)
            .queryParam("startCreateDt", "20200915")
            .queryParam("endCreateDt", "20200916")
            .build(true);
    URI uri = uriComponents.toUri();

    String result = restTemplate.getForObject(uri, String.class);
}
```



### WebClient
`WebClient`는 리액티브 스택 기반의 웹 애플리케이션에서 사용할 수 있도록 [`spring-webflux`](https://mvnrepository.com/artifact/org.springframework/spring-webflux/5.2.8.RELEASE) 모듈에서 제공하는 논-블로킹 HTTP 요청 클라이언트입니다. WebClient는 기본적으로 [`Reactor Netty`](https://github.com/reactor/reactor-netty)를 HTTP 클라이언트 라이브러리를 사용하게 됩니다.

WebClient는 논-블로킹 HTTP 요청 클라이언트이지만 동기 방식도 지원합니다. 동기 방식을 지원하므로 서블릿 스택 기반의 웹 애플리케이션에서 WebClient를 사용할 수도 있습니다. 서블릿 스택 기반의 웹 애플리케이션에서 WebClient를 사용하고 싶다면 `spring-webflux`와 함께 [`reactor-netty`](https://mvnrepository.com/artifact/io.projectreactor.netty/reactor-netty/0.9.11.RELEASE) 의존성을 추가합니다.

```groovy build.gradle
implementation 'org.springframework:spring-webflux:5.2.8.RELEASE'
implementation 'io.projectreactor.netty:reactor-netty:0.9.11.RELEASE'
```

WebClient는 RestTemplate와 비교하여 다음과 같이 사용할 수 있습니다.
```java
private static final String SERVICE_KEY = "";
@Test
public void TEST_002() throws IOException, SAXException, ParserConfigurationException {
    DefaultUriBuilderFactory uriBuilderFactory = new DefaultUriBuilderFactory("http://openapi.data.go.kr/openapi/service/rest/Covid19/getCovid19InfStateJson");
    uriBuilderFactory.setEncodingMode(DefaultUriBuilderFactory.EncodingMode.VALUES_ONLY);

    WebClient webClient = WebClient.builder()
            .uriBuilderFactory(uriBuilderFactory)
            .build();

    String result = webClient.get()
            .uri(uriBuilder -> {
                uriBuilder
                    .queryParam("serviceKey", SERVICE_KEY)
                    .queryParam("pageNo", 1)
                    .queryParam("numOfRows", 10)
                    .queryParam("startCreateDt", "20200915")
                    .queryParam("endCreateDt", "20200916");
                return uriBuilder.build();
            })
            .exchange()
            .block(Duration.ofMinutes(1))
            .bodyToMono(String.class)
            .block();
}
```
