---
title: 스프링 프레임워크는 어떻게 예외를 처리할까?
date: 2020-09-25
categories:
- 스프링 프레임워크 이야기
---

## Overview
스프링 프레임워크 기반의 웹 애플리케이션에서 웹 요청을 처리하는 과정에서 발생하는 예외(Exception)에 대하여 어떻게 처리하는 걸까요? 이번 글에서는 스프링 프레임워크가 예외에 대한 처리를 담당하는 인터페이스와 함께 여러가지 예외 처리 전략에 대하여 알아봅니다.

## Exception Handling Strategy
이번 글의 목표는 스프링 프레임워크가 요청에 대한 예외를 처리하는 전략에 대해서 알아보는 것입니다. 스프링 프레임워크의 `webmvc` 모듈에는 `HandlerExceptionResolver`라는 인터에피스를 제공합니다. HandlerExceptionResolver는 웹 요청을 처리할 핸들러 함수를 매핑하거나 핸들러 함수가 요청을 처리하는 과정에서 발생하는 예외를 처리할 수 있도록 지원합니다.

```java
public interface HandlerExceptionResolver {
    @Nullable
    ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);
}
```

다시 말해, HandlerExceptionResolver는 디스패처 서블릿과 컨트롤러 컴포넌트에서 발생하는 예외에 대하여 어떻게 처리해야하는지를 결정하는 전략이라고 할 수 있습니다. 우리는 이 전략에 대하여 공부하기 위하여 HandlerExceptionResolver 구현체들을 살펴보도록 합니다.

### HandlerExceptionResolver
`webmvc` 모듈에는 스프링 프레임워크에서 제공하는 기본적인 HandlerExceptionResolver 구현체들이 포함되어있습니다. 

- HandlerExceptionResolverComposite
- ExceptionHandlerExceptionResolver
- ResponseStatusExceptionResolver
- DefaultHandlerExceptionResolver

특히, `HandlerExceptionResolverComposite` 구현체는 `@EnableWebMvc`를 구성 메타정보 클래스에 선언하면 추가되는 `WebMvcConfigurationSupport` 클래스에서 HandlerExceptionResolver 유형의 빈으로 등록합니다. 이 구현체는 다른 HandlerExceptionResolver 구현체들에게 예외에 대한 처리를 위임하도록 담당하는 특별한 역할을 하게 됩니다. 따라서, 우리는 HandlerExceptionResolverComposite를 통해 여러가지 HandlerExceptionResolver 구현체를 통해 예외를 처리할 수 있도록 전략을 적용할 수 있게 됩니다.


```java HandlerExceptionResolverComposite.java
@Override
@Nullable
public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
    if (this.resolvers != null) {
        for (HandlerExceptionResolver handlerExceptionResolver : this.resolvers) {
            ModelAndView mav = handlerExceptionResolver.resolveException(request, response, handler, ex);
                if (mav != null) {
                    return mav;
                }
            }
        }
    return null;
}
```

우리는 이제 스프링 프레임워크에서 제공하면서 예외에 대한 처리를 담당하는 HandlerExceptionResolver 구현체에 대해서 알아볼 것입니다. HandlerExceptionResolverComposite는 예외 처리를 담당할 HandlerExceptionResolver를 다음과 같은 순서로 처리하게 됩니다.

- ExceptionHandlerExceptionResolver
- ResponseStatusExceptionResolver
- DefaultHandlerExceptionResolver

HandlerExceptionResolverComposite의 `addDefaultHandlerExceptionResolvers()` 함수를 살펴보면 위 순서로 등록하는 것을 알 수 있습니다.

```java HandlerExceptionResolverComposite.java
protected final void addDefaultHandlerExceptionResolvers(List<HandlerExceptionResolver> exceptionResolvers, ContentNegotiationManager mvcContentNegotiationManager) {
    ExceptionHandlerExceptionResolver exceptionHandlerResolver = createExceptionHandlerExceptionResolver();
    exceptionHandlerResolver.setContentNegotiationManager(mvcContentNegotiationManager);
    exceptionHandlerResolver.setMessageConverters(getMessageConverters());
    exceptionHandlerResolver.setCustomArgumentResolvers(getArgumentResolvers());
    exceptionHandlerResolver.setCustomReturnValueHandlers(getReturnValueHandlers());
    if (jackson2Present) {
        exceptionHandlerResolver.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
    }
    if (this.applicationContext != null) {
        exceptionHandlerResolver.setApplicationContext(this.applicationContext);
    }
    exceptionHandlerResolver.afterPropertiesSet();
    exceptionResolvers.add(exceptionHandlerResolver);

    ResponseStatusExceptionResolver responseStatusResolver = new ResponseStatusExceptionResolver();
    responseStatusResolver.setMessageSource(this.applicationContext);
    exceptionResolvers.add(responseStatusResolver);

    exceptionResolvers.add(new DefaultHandlerExceptionResolver());
}
```

#### ExceptionHandlerExceptionResolver
HandlerExceptionResolverComposite에 의해 가장 먼저 예외를 처리하는 `ExceptionHandlerExceptionResolver`는 `@ExceptionHandler`가 선언된 핸들러 함수에서 발생한 예외를 담당합니다.

```java ExceptionHandlerExceptionResolver.java
public class ExceptionHandlerExceptionResolver extends AbstractHandlerMethodExceptionResolver implements ApplicationContextAware, InitializingBean {
    @Override
    @Nullable
    protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request, HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {
        ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
        if (exceptionHandlerMethod == null) {
            return null;
        }
        // ...
    }
}
```

위 코드처럼 `@ExceptionHandler`가 선언된 핸들러 함수라면 예외에 대한 처리를 담당하며 아니라면 null을 반환하여 다른 HandlerExceptionResolver 구현체가 예외에 대한 처리를 담당하도록 합니다. ExceptionHandlerExceptionResolver는 ServletInvocableHandlerMethod를 활용하여 핸들러 함수를 호출합니다. 이 과정에서 HandlerMethodArgumentResolver와 HandlerMethodReturnValueHandler를 통해 핸들러 함수의 매개변수에 값을 주입하거나 응답 객체에 따른 응답을 지원합니다. 다시 말하면, @ExceptionHandler가 선언된 핸들러 함수는 컨트롤러 컴포넌트에 선언된 핸들러 함수와 같이 동작하는 것이라고 할 수 있습니다.

따라서, `@ExceptionHandler`는 `@Controller` 또는 `@ControllerAdvice`가 선언된 컴포넌트의 함수에 선언할 수 있습니다.

#### ResponseStatusExceptionResolver
ExceptionHandlerExceptionResolver에 의해 처리되지 않은 예외라면 @ExceptionHandler가 선언된 핸들러 함수에서 해당 예외 클래스를 담당하지 않았다는 것입니다. 이 예외들은 `ResponseStatusExceptionResolver`가 처리를 담당하게 되며 예외 클래스에 `@ResponseStatus`가 선언되었는지를 판단하여 처리할 지 결정하게 됩니다. 

```java ResponseStatusExceptionResolver.java
@Override
@Nullable
protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

    try {
        if (ex instanceof ResponseStatusException) {
            return resolveResponseStatusException((ResponseStatusException) ex, request, response, handler);
        }

        ResponseStatus status = AnnotatedElementUtils.findMergedAnnotation(ex.getClass(), ResponseStatus.class);
        if (status != null) {
            return resolveResponseStatus(status, request, response, handler, ex);
        }

        if (ex.getCause() instanceof Exception) {
            return doResolveException(request, response, handler, (Exception) ex.getCause());
        }
    }
    catch (Exception resolveEx) {
        if (logger.isWarnEnabled()) {
            logger.warn("Failure while trying to resolve exception [" + ex.getClass().getName() + "]", resolveEx);
        }
    }
    return null;
}
```

스프링 5+ 부터는 `ResponseStatusException` 예외 클래스가 추가되었으며 ResponseStatusExceptionResolver가 ResponseStatusException 클래스도 담당하여 처리하게 됩니다. 그리고 다른 예외 클래스에 @ResponseStatus가 선언되었다면 설정된 값에 의해 예외를 처리하게 됩니다. 따라서, 우리는 예외 클래스에 @ResponseStatus를 선언하여 쉽게 예외에 대한 응답을 처리할 수 있습니다.

#### DefaultHandlerExceptionResolver
가장 마지막으로 처리를 담당하는 `DefaultHandlerExceptionResolver`는 스프링 프레임워크에서 제공하는 기본 MVC 예외 클래스에 대한 예외 처리를 담당합니다. 다음은 DefaultHandlerExceptionResolver가 처리하는 예외 유형과 HTTP 상태 코드에 대한 표입니다.

|Exception|HTTP Status Code|Description|
|---|---||
|HttpRequestMethodNotSupportedException|405 (SC_METHOD_NOT_ALLOWED)|HTTP 메소드를 지원하지 않는 경우|
|HttpMediaTypeNotSupportedException|415 (SC_UNSUPPORTED_MEDIA_TYPE)|POST, PUT, PATCH 요청에 대하여 지원하지 않는 미디어 타입인 경우|
|HttpMediaTypeNotAcceptableException|406 (SC_NOT_ACCEPTABLE)||
|MissingPathVariableException|500 (SC_INTERNAL_SERVER_ERROR)|URI 템플릿에 PathVariable을 찾을 수 없는 경우|
|MissingServletRequestParameterException|400 (SC_BAD_REQUEST)|요청 파라미터를 찾을 수 없는 경우|
|ServletRequestBindingException|400 (SC_BAD_REQUEST)|바인딩 과정에서 오류가 발생한 경우|
|ConversionNotSupportedException|500 (SC_INTERNAL_SERVER_ERROR)||
|TypeMismatchException|400 (SC_BAD_REQUEST)||
|HttpMessageNotReadableException|400 (SC_BAD_REQUEST)||
|HttpMessageNotWritableException|500 (SC_INTERNAL_SERVER_ERROR)|HttpMessageConverter에 의해 변환될 수 없는 경우|
|MethodArgumentNotValidException|400 (SC_BAD_REQUEST)||
|MissingServletRequestPartException|400 (SC_BAD_REQUEST)||
|BindException|400 (SC_BAD_REQUEST)||
|NoHandlerFoundException|404 (SC_NOT_FOUND)|throwExceptionIfNoHandlerFound이면서 웹 요청에 대한 핸들러 함수가 없는 경우|
|AsyncRequestTimeoutException|503 (SC_SERVICE_UNAVAILABLE)||

만약, 스프링 프레임워크에서 제공하는 기본 예외 클래스들에 대한 응답 전략 구성하고 싶다면 @ExceptionHandler를 선언한 핸들러 함수에서 처리할 수 있습니다. 그리고 DefaultHandlerExceptionResolver는 기본 예외 클래스에 대한 응답 전략을 제공하므로 HandlerExceptionResolver 유형의 빈으로 등록하는 것이 좋습니다.

## Examples
이 섹션은 앞서 알아본 예외 처리 전략에 대해서 공부할 수 있는 예시를 제공합니다.

> 업데이트 대기중

---

이번 글을 통해 우리는 스프링 프레임워크 기반의 웹 애플리케이션에서 요청에 대한 예외를 어떻게 처리하는지를 알게되었습니다. 이제 여러분의 프로젝트에서는 어떤 예외 처리 전략을 구성하였는지 확인해보시고 올바르게 전략을 세우고 있는지 검토해보시기 바랍니다.

