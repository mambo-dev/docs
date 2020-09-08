---
title: Spring Framework Documentation
date: 2020-09-08
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## 들어가며
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다. 스프링 애플리케이션 개발 시 프레임워크 버전을 확인하여 참고하시기 바랍니다.

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



