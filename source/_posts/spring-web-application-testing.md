---
title: 스프링 웹 애플리케이션 - 단위 테스트
date: 2020-09-13
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

스프링 프레임워크 기반의 웹 애플리케이션을 개발하기 위해서 매번 애플리케이션을 실행하는 것은 불편함이 있습니다. 전체적인 웹 애플리케이션의 동작을 확인하는 것이 아니라면 개발하는 로직의 단위 테스트를 수행하여 개발하는 것이 좋습니다.

스프링 프레임워크는 애플리케이션 테스트를 위한 [`spring-test`](https://mvnrepository.com/artifact/org.springframework/spring-test/5.2.8.RELEASE) 모듈을 제공합니다.

이번 글에서는 스프링 테스트 모듈을 통해 간단하게 단위 테스트를 수행하는 것에 대해 알아봅니다. 테스트에 대한 자세한 내용은 [공식 레퍼런스](https://docs.spring.io/spring/docs/5.2.8.RELEASE/spring-framework-reference/testing.html)를 참고하세요.

## Unit Testing
스프링 테스트 모듈을 활용하면 서비스 계층 클래스를 테스트 및 개발을 쉽게 수행할 수 있습니다. 

```groovy build.gradle
testCompile 'org.springframework:spring-test:5.2.8.RELEASE'
testCompile group: 'junit', name: 'junit', version: '4.12'
```

### Spring JUnit 4 Runner
스프링 테스트 컨테스트 프레임워크는 JUnit4으로 테스트를 수행할 수 있도록 SpringJUnit4ClassRunner를 제공합니다.

`@RunWith`를 선언하여 JUnit으로 테스트를 수행할 `SpringJUnit4ClassRunner`를 지정할 수 있습니다.
```java
@RunWith(SpringRunner.class)
public class SimpleTest {

    @Test
    public void testMethod() {
        // test logic...
    }
}
```

### Testing Annotations
스프링 테스트 모듈은 애플리케이션 테스트를 위한 어노테이션을 제공합니다.

다음 예제는 스프링 테스트 모듈이 제공하는 어노테이션으로 이메일 발송에 대한 테스트 코드를 작성한 것입니다.
```java
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = {MailConfig.class})
@ActiveProfiles({"test"})
@TestPropertySource(properties = { "mail.username = username@gmail.com", "mail.password: password" })
public class MailServiceTests {

    @Autowired
    private MailService mailService;

    @Test
    public void test() throws MessagingException, IOException, TemplateException {
        Map<String, Object> variables = new HashMap<>();
        variables.put("name", "Mambo");
        variables.put("content", "This mail is sent for testing.");
        MimeMessage mimeMessage = mailService.getMimeMessage("username@gmail.com", "Send test mail", variables);
        boolean status = mailService.sendMail(mimeMessage);
    }
}
```

- @ContextConfiguration는 테스트 컨텍스트를 구성하기 위한 메타정보 클래스를 지정할 수 있습니다.
- @ActiveProfiles는 테스트 컨텍스트에 적용할 프로파일 지정할 수 있습니다.
- @TestPropertySource는 테스트 컨텍스트에 적용되는 프로퍼티를 임의로 설정할 수 있도록 지원합니다.
- @DirtiesContext는 테스트가 실행되는 동안 변경되는 테스트 컨텍스트에 대한 캐시를 제거할 수 있는 방법을 지원합니다.
- @ContextHierarchy는 테스트 컨텍스트에 대해 계층 구조를 갖게 할 수 있습니다.

## Spring MVC Test
스프링 테스트 모듈에는 스프링 웹 MVC 테스트를 위한 기능이 포함되어 있습니다. `JUnit` 또는 `TestNG`를 사용하여 컨트롤러에 대한 테스트를 작성할 수 있습니다.

### @WebAppConfiguration
`@WebAppConfiguration`을 선언하여 테스트를 위한 웹 애플리케이션 컨텍스트를 구성할 수 있습니다.

```java
@RunWith(SpringRunner.class)
@ContextHierarchy({
    @ContextConfiguration(classes = {AppConfig.class}),
    @ContextConfiguration(classes = {WebConfig.class})
})
@WebAppConfiguration
public class SpringApplicationTests {

    @Autowired
    protected WebApplicationContext wac;

    @Test
    public void contextLoads() {
        System.out.println(wac);
    }
}
```

### MockMvc
스프링 테스트 모듈에는 `Servlet API`에 대한 모의 구현체가 포함되어 있습니다. `모의 구현체`를 활용하여 웹 애플리케이션 테스트를 수행할 수 있습니다.

```java
@RunWith(SpringRunner.class)
@ContextHierarchy({
    @ContextConfiguration(classes = {AppConfig.class}),
    @ContextConfiguration(classes = {WebConfig.class})
})
@WebAppConfiguration
public class SpringApplicationTests {

    @Autowired
    protected WebApplicationContext wac;
    protected MockMvc mockMvc;

    @Before
    public void setup() {
        mockMvc = MockMvcBuilders.webAppContextSetup(wac).build();
    }

    @Test
    public void contextLoads() {
        System.out.println(mockMvc);
    }
}
```

---

스프링 프레임워크에서 제공하는 스프링 테스트 모듈을 사용하면 단위 테스트 뿐만 아니라 웹 애플리케이션을 구동하지 않아도 테스트를 위한 애플리케이션 컨텍스트를 구성할 수 있음을 확인하였습니다. 


