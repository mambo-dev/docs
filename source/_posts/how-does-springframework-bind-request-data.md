---
title: 스프링 프레임워크는 어떻게 요청 데이터를 바인딩할까?
date: 2020-09-17
categories:
- 스프링 프레임워크 이야기
---

## Overview
스프링 프레임워크 이야기는 스프링 애플리케이션 개발자들이 궁금해할만한 내용에 대하여 잠만보가 찾아보고 정리하는 글입니다.

스프링 프레임워크의 컨트롤러 컴포넌트를 작성할 때 핸들러 함수에서 사용하는 모델 오브젝트에 대해 `@ModelAttribute`와 `@RequestBody`를 선언하여 사용하시나요? 그렇다면 @ModelAttribute와 @RequestBody를 어느 상황에서 사용해야하는지 알고 계신가요? 만약, 이 질문에 대답을 못하였다면 이 글을 읽는 것이 도움이 될 수 있습니다.

## Data Binding
오늘의 주제는 데이터 바인딩입니다. 데이터 바인딩이란 제공받은 데이터를 내가 보유한 객체에 주입하는 것을 말합니다. 스프링 프레임워크에서 데이터 바인딩은 [`JavaBeans Specification`](https://www.oracle.com/java/technologies/javase/javabeans-spec.html)를 기반으로 데이터를 자바 빈 오브젝트에 넣을 수 있는 기능을 제공합니다.

스프링 프레임워크의 `spring-beans` 모듈에 포함되어있는 `BeanWrapper` 인터페이스는 자바 빈 스펙을 기초로하여 오브젝트의 내부 프로퍼티를 설정할 수 있는 기능을 제공합니다. 다만, 이 BeanWrapper는 애플리케이션 코드에서 직접적으로 사용하지않으며 `DataBinder` 또는 `BeanFactory`에 의해 사용됩니다.

### Setting and Getting Basic and Nested Properties
BeanWrapper를 통해 프로퍼티를 설정하고 가져올때 `setPropertyValue`와 `getPropertyValue`를 사용하게 됩니다. 그리고 스프링 공식 레퍼런스에서는 다음과 같은 프로퍼티를 가져오고 설정하는 규칙에 대해서 설명합니다.

![](/images/posts/spring-story-001.png)

### Binding Annotations
스프링 프레임워크에는 데이터 바인딩을 위한 여러가지 어노테이션이 있습니다. 스프링 웹 애플리케이션 개발자는 웹 요청을 처리할 컨트롤러 핸들러 함수를 작성할 때 `@RequestParam` 또는 `@ModelAttribute`, `@PathVariable`, `@RequestBody`와 같은 어노테이션을 주로 사용합니다.

컨트롤러 핸들러 함수 파라미터에 위와 같은 바인딩 어노테이션이 선언되면 RequestMappingHandlerAdapter에 등록된 HandlerMethodArgumentResolver 구현체에 의해 데이터 바인딩을 수행합니다.

- ServletRequestMethodArgumentResolver
- ServletModelAttributeMethodProcessor
- RequestParamMethodArgumentResolver
- PathVariableMethodArgumentResolver
- ServletCookieValueMethodArgumentResolver
- RequestResponseBodyMethodProcessor

### @ModelAttribute vs @RequestBody
이번 데이터 바인딩에 대한 주 비교 대상은 @ModelAttribute와 @RequestBody가 선언된 파라미터에 대해 데이터 바인딩을 시도하는 `ServletModelAttributeMethodProcessor`와 `RequestResponseBodyMethodProcessor`입니다.

가끔씩 스프링 웹 애플리케이션 개발자들이 이러한 질문을 하곤 합니다. 

> @ModelAttribute를 선언했는데 핸들러 함수 파라미터에 값이 `null`이에요.

@ModelAttribute를 선언했지만 바인딩된 데이터가 없다는 뜻인데 왜 이러한 결과가 나타나는 걸까요? 이 이유를 찾기 위해 @ModelAttribute가 선언된 파라미터에 대한 데이터 바인딩을 담당하는 ServletModelAttributeMethodProcessor에 대하여 살펴보도록 하죠.

스프링 프레임워크가 제공하는 API 문서를 보면 [`ServletModelAttributeMethodProcessor`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ServletModelAttributeMethodProcessor.html)에 대해서 다음과 같이 설명하고 있습니다.

> A Servlet-specific ModelAttributeMethodProcessor that applies data binding through a WebDataBinder of type ServletRequestDataBinder.
> Also adds a fall-back strategy to instantiate the model attribute from a URI template variable or from a request parameter if the name matches the model attribute name and there is an appropriate type conversion strategy.

ServletRequestDataBinder 유형의 WebDataBinder를 통해서 데이터 바인딩을 적용한다고 합니다. 그리고 두번째 문장을 통해 URI 템플릿 변수와 요청 파라미터에 대해 특정 전략으로 수행한다는 것을 알 수 있습니다. 

다시 정리해보면 `@ModelAttribute`가 선언된 매개변수는 `URI 템플릿 변수` 또는 `요청 파라미터(Request Parameter)`에 있는 데이터를 바인딩할 수 있다고 이해할 수 있겠네요.

해당 질문에서 다음과 같이 웹 요청을 시도하고 있다고 했습니다.

```
HTTP Method: POST
Content-Type: application/json
Data: Request Body
```

POST 요청이고 데이터는 요청 바디에 있으면서 데이터 유형은 JSON 형식이네요. 눈치가 빠르신분들은 바로 알아차렸을 겁니다. 

> 데이터는 요청 바디에 포함하여 전송된다...

앞서 @ModelAttribute가 선언된 매개변수는 요청 파라미터에 있는 데이터를 바인딩하는 것입니다. 그런데 질문했던 개발자는 데이터를 요청 바디에 포함하여 웹 요청을 하고 있었습니다. 그리고 답변을 달았던 다른 개발자는 이유를 설명하지 않고 @ModelAttribute 를 `@RequestBody`로 바꾸라고 하였습니다.

우선 첫번째 질문에 대한 원인은 알았으니 @RequestBody가 선언된 매개변수에 대한 데이터 바인딩은 어떻게 수행하는지 알아보도록 합시다. `@RequestBody`가 선언된 매개변수는 `RequestResponseBodyMethodProcessor`가 데이터 바인딩을 담당합니다. [`RequestResponseBodyMethodProcessor`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/RequestResponseBodyMethodProcessor.html)에 대해서는 다음과 같이 설명합니다.

> Resolves method arguments annotated with @RequestBody and handles return values from methods annotated with @ResponseBody by reading and writing to the body of the request or response with an HttpMessageConverter.

위 문장에서 @RequestBody에 대한 부분만 정리하면 `@RequestBody`가 선언된 매개변수는 `HttpMessageConverter`에 의해 요청 바디를 읽는다고 할 수 있습니다. 그리고 application/json 유형의 요청 바디를 변환할 수 있는 HttpMessageConverter는 다음과 같습니다.

- MappingJackson2HttpMessageConverter - `jackson-databind`
- GsonHttpMessageConverter - `gson`

### Form Data
브라우저에서는 폼 데이터를 전송할 때 POST를 사용합니다. [HTML 폼 데이터를 전송하는 방식](https://developer.mozilla.org/ko/docs/Learn/HTML/Forms/Sending_and_retrieving_form_data)을 찾아보면 `application/x-www-form-urlencoded` 형식의 데이터를 요청 바디에 포함하여 전송되는 것을 확인할 수 있습니다. 

그러나 브라우저가 아닌 웹 요청 클라이언트는 PUT, DELETE도 사용할 수 있습니다. 그런데 [스프링 공식 레퍼런스](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#filters-http-put)에 따르면 서블릿 API는 ServletRequest.getParameter*() 함수가 HTTP POST일 때만 폼 필드에 접근할 수 있다고 합니다. 

위와 같은 내용이 맞다면 웹 요청 클라이언트가 PUT을 이용하여 요청 바디를 `application/x-www-form-urlencoded` 형식으로 전송한다면 스프링 웹 프레임워크는 요청 바디를 @ModelAttribute를 사용하여 바인딩할 수 없습니다. 그래서 스프링 프레임워크는 PUT이나 PATCH, DELETE 요청에 대해서도 `application/x-www-form-urlencoded`로 전송되는 요청 바디를 읽어 `ServletRequest.getParameter*()`로 접근할 수 있도록 지원하는 `FormContentFilter`를 제공합니다.

스프링 프레임워크의 FormContentFilter에서 사용하는 `FormHttpMessageConverter`는 `application/x-www-form-urlencoded`로 전송되는 데이터를 `Form Data`로 변환하는 작업을 수행합니다. 결국 우리는 @ModelAttribute를 선언하여 데이터 바인딩이 가능해집니다.

#### @RequestBody
`x-www-form-urlencoded` 형식으로 전송되는 요청 바디는 @RequestBody로 읽을 수 없습니다. 그래서 다음과 같은 오류가 발생합니다.

```
HTTP Status 415 – Unsupported Media Type
```

왜 메시지 컨버터를 통해 `x-www-form-urlencoded` 형식으로 전송한 요청 바디를 모델 오브젝트에 바인딩할 수 없었을까요?

스프링 웹 MVC는 기본적으로 FormHttpMessageConverter를 메시지 컨버터로 등록합니다. 그러니까 @RequestBody로 선언되어있고 `application/x-www-form-urlencoded`로 전송되었다면 FormHttpMessageConverter를 통해 메시지 컨버팅을 시도합니다. 그러나 FormHttpMessageConverter에 대한 설명을 찾아보면 다음과 같습니다.

> this converter can read and write the "application/x-www-form-urlencoded" media type as MultiValueMap<String, String>, and it can also write (but not read) the "multipart/form-data" and "multipart/mixed" media types as MultiValueMap<String, Object>.

FormHttpMessageConverter는 `application/x-www-form-urlencoded` 전송인 경우 `MultiValueMap<String, String>` 형식으로 변환하며 `multipart/form-data`로 전송되는 경우 `MultiValueMap<String, Object>`로 변환된다고 설명합니다.

따라서, 모델 오브젝트에 대해서 @RequestBody로 데이터 바인딩을 시도할 때 `application/x-www-form-urlencoded`로 전송된 데이터는 MultiValueMap<String, String> 형식의 오브젝트로만 변환된다는 말입니다. 결국 FormHttpMessageConverter가 처리할 수 없는 유형이기 때문에 Unsupported Media Type가 응답된 것입니다.

---

앞선 내용을 종합해서 정리하면 다음과 같습니다.

1. HTML 폼 데이터 전송은 `application/x-www-form-urlencoded`으로 보내는 것이다.
2. 기본 서블릿 API는 POST로 전송되었을때만 폼 데이터에 접근할 수 있도록 `제한`한다.
3. 스프링 프레임워크는 `PUT`, `PATCH`, `DELETE`와 함께 application/x-www-form-urlencoded로 전송되었을때 폼 데이터에 접근할 수 있도록 래핑한다.
4. @RequestBody가 선언되고 application/x-www-form-urlencoded로 전송된 경우 `FormHttpMessageConverter`에 의해 `MultiValueMap`으로 변환된다.
5. application/x-www-form-urlencoded로 전송되는 데이터를 `@RequestBody`를 선언하여 모델 오브젝트로 바인딩할 수 없다.

이제 여러분은 컨트롤러 핸들러 함수의 매개변수에 대한 데이터 바인딩을 위해 @ModelAttribute를 사용해야할 경우와 @RequestBody를 사용해야하는 경우를 구분할 수 있게 되었습니다. 부디 많은 스프링 애플리케이션 개발자들에게 도움이 되었길 바랍니다.