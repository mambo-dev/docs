---
title: 스프링 웹 애플리케이션 - 보안
date: 2020-09-18
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
본 글은 스프링 시큐리티 Version `5.3.4.RELEASE` 문서와 [`초보가 이해하는 스프링 시큐리티`](https://okky.kr/article/382738)를 기반으로 작성하였습니다.

## Spring Security
보안 전문가가 아니고서야 일반 개발자가 애플리케이션에 대한 보안 기능을 적용하는 것은 쉽지 않습니다. 스프링 시큐리티는 스프링 프레임워크를 기반으로 애플리케이션을 개발하는 개발자들이 쉽게 보안을 적용할 수 있는 매커니즘을 제공합니다. 

### Security Architecture
스프링 프레임워크 기반의 애플리케이션에 스프링 시큐리티를 적용하기에 앞서 스프링 시큐리티의 기반이 되는 보안 아키텍처에 대해 알아야합니다. 저는 이전에 [초보가 이해하는 스프링 시큐리티](https://okky.kr/article/382738)에서 인증과 권한에 개념을 설명하였습니다. 

> 인증이란 애플리케이션의 작업을 수행할 수 있는 주체(사용자)라고 주장하는 것이며 권한은 인증된 주체가 애플리케이션의 동작을 수행할 수 있도록 허락하는 것이다.

따라서, 스프링 시큐리티의 보안 아키텍처에서 애플리케이션 사용자는 애플리케이션에 대한 작업을 수행할 수 있는 주체라고 인증하는 것, 애플리케이션은 인증된 주체에 대하여 권한을 제어하는 것을 통해 보안 기능을 적용합니다.

### springSecurityFilterChain
스프링 시큐리티는 `springSecurityFilterChain`라는 이름의 특별한 필터를 통해 보안을 적용합니다. `@EnableWebSecurity`을 구성 메타정보 클래스에 선언하면 `springSecurityFilterChain` 이름의 필터가 빈으로 등록됩니다.

```java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig {}
```

그리고 이 보안 필터를 다른 필터가 수행되기 이전에 처리될 수 있도록 지원하는 DelegatingFilterProxy를 등록하기 위하여 AbstractSecurityWebApplicationInitializer를 클래스 패스에 생성합니다.

```java
public class WebSecurityApplicationInitializer extends AbstractSecurityWebApplicationInitializer {}
```

이제 스프링 프레임워크 기반의 웹 애플리케이션을 실행하면 기본적인 웹 보안이 적용됩니다. 따라서, http://localhost:8080 으로 접근하게 되면 접근이 금지되고 인증을 위한 페이지인 `/login`으로 이동됩니다.

![](https://docs.spring.io/spring-security/site/docs/5.3.4.RELEASE/reference/html5/images/servlet/architecture/filterchainproxy.png)

위 그림은 FilterChainProxy에 의해 springSecurityFilterChain으로 필터 과정이 위임되는 것을 보여주고 있습니다. 앞선 요청은 위 과정에 의해 springSecurityFilterChain에 등록된 DefaultLoginPageGeneratingFilter에 의해 인증을 위한 로그인 페이지로 이동된 것입니다.

#### Filter Debugging
springSecurityFilterChain에 의해 수행되는 필터와 처리되는 순서를 알고 싶다면 @EnableWebSecurity를 선언할 때 디버그 모드를 활성화하면 됩니다.

```java
@EnableWebSecurity(debug = true)
@Configuration
public static class WebSecurityConfig extends WebSecurityConfigurerAdapter {}
```

디버그 모드가 활성화되면 `Spring Security Debugger`에 의해 다음과 같이 요청 및 처리되는 필터 순서가 출력됩니다.
```
[http-nio-8080-exec-1] INFO  Spring Security Debugger(54) - 

************************************************************

Request received for GET '/':

org.springframework.session.web.http.SessionRepositoryFilter$SessionRepositoryRequestWrapper@353695df

servletPath:/
pathInfo:null
headers: 
host: localhost:8080
connection: keep-alive
cache-control: max-age=0
upgrade-insecure-requests: 1
user-agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36
accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
sec-fetch-site: none
sec-fetch-mode: navigate
sec-fetch-user: ?1
sec-fetch-dest: document
accept-encoding: gzip, deflate, br
accept-language: ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7
cookie: SESSION=YTE0ZTUxODUtMzg0NS00MGMwLWFkOGYtYzFiNDFmYjdkYTMy; io=qmiiYVyd26T7GqDfAABF


Security filter chain: [
  WebAsyncManagerIntegrationFilter
  SecurityContextPersistenceFilter
  HeaderWriterFilter
  CsrfFilter
  LogoutFilter
  RequestCacheAwareFilter
  SecurityContextHolderAwareRequestFilter
  AnonymousAuthenticationFilter
  SessionManagementFilter
  ExceptionTranslationFilter
]


************************************************************
```

## Security Configuration
스프링 프레임워크 기반의 웹 애플리케이션에 보안 기능이 적용되었습니다. 그러나, 적용된 보안은 애플리케이션이 요구하는 보안 사항이 아닐 수 있습니다. 그러므로 우리는 스프링 시큐리티의 보안 구성을 변경하여 우리만의 보안을 적용해야합니다.

### Authentication Mechanisms
먼저, 애플리케이션에 대한 인증 매커니즘을 구성하는 것부터 시작합니다. 가장 기본적인 인증 프로세스는 폼 로그인입니다. 다시 말해, 사용자 이름과 비밀번호를 제공받아 사용자를 인증하는 것입니다.

![](https://docs.spring.io/spring-security/site/docs/5.3.4.RELEASE/reference/html5/images/servlet/authentication/unpwd/usernamepasswordauthenticationfilter.png)

위 그림은 스프링 시큐리티 공식 레퍼런스에서 제공하는 폼 로그인이 수행되는 과정을 설명하는 그림입니다. `springSecurityFilterChain`에 등록된 `UsernamePasswordAuthenticationFilter`에 의해 `UsernamePasswordAuthenticationToken`을 생성하여 `AuthenticationManager`에 의해 인증 여부를 판단하고 있습니다. 만약, 인증에 실패했다면 SecurityContextHolder가 초기화되고 AuthenticationFailureHandler가 호출되고 인증에 성공했다면 SecurityContextHolder에 인증 정보가 설정되고 AuthenticationSuccessHandler가 호출됩니다.

다음은 DefaultLoginPageGeneratingFilter를 대신하여 폼 로그인을 구성하는 예시입니다.
```java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin(form -> {
            form.loginPage("/login")
                .usernameParameter("id")
                .passwordParameter("password");
        });
    }
}
```

단, DefaultLoginPageGeneratingFilter이 등록되지 않으므로 `/login` 경로에 대한 컨트롤러 핸들러 함수를 등록하여야합니다.

#### In-Memory Authentication
메모리 기반 인증은 가장 간단한 인증 매커니즘입니다. `InMemoryUserDetailsManager`는 인증 정보를 메모리에 구성할 수 있도록 지원합니다.

```java
@Bean
public UserDetailsService userDetailsService() {
    UserDetails admin = User.builder()
            .username("admin")
            .password("$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
            .roles("ADMIN")
            .build();

    InMemoryUserDetailsManager userDetailsManager = new InMemoryUserDetailsManager(admin);

    UserDetails user = User.builder()
            .username("user")
            .password("$2a$10$GRLdNijSQMUvl/au9ofL.eDwmoohzzS7.rmNSJZ.0FxO/BTk76klW")
            .roles("USER")
            .build();
    userDetailsManager.createUser(user);

    return userDetailsManager;
}
```

#### Database Authentication
`UserDetailsService` 구현체를 만들어서 저장소 기반의 인증 매커니즘을 구성할 수 있습니다. 스프링 시큐리티에는 메모리 기반 인증을 위한 `InMemoryUserDetailsManager` 또는 JDBC 기반 인증을 위한 `JdbcUserDetailsManager` 구현체를 제공합니다. 다만, JdbcUserDetailsManager을 위한 테이블 스키마가 정해져있으므로 직접 구성되는 데이터베이스 스키마에 따라 직접 `UserDetailsService` 구현체를 만드는 것이 좋습니다.

UserDetailsService 구현체는 `UserDetails`를 반환함으로써 인증 정보를 제공합니다.
```java
public interface UserDetailsService {
	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

우리는 `loadUserByUsername()`을 오버라이딩하여 저장소에서 정보를 가져와 UserDetails를 반환하도록 코드를 작성하면 됩니다.

#### AuthenticationProvider
UserDetailsService 구현체에 의해 구성된 인증 매커니즘으로 인증을 수행되는 것에 궁금해할 수 있습니다. 스프링 시큐리티는 인증 매커니즘에 의해 인증을 수행하는 것을 `AuthenticationProvider` 구현체가 담당합니다. 구현체 중 하나인 [`DaoAuthenticationProvider`](https://docs.spring.io/spring-security/site/docs/5.3.4.RELEASE/api/org/springframework/security/authentication/dao/DaoAuthenticationProvider.html)는 AuthenticationProvider 구현체로 UserDetailsService와 PasswordEncoder를 통해 인증을 수행하게 됩니다.

### Configure HttpSecurity

#### Authorize Requests
`ExpressionUrlAuthorizationConfigurer`는 URL 패턴에 대하여 보안을 적용할 수 있도록 지원합니다.

다음은 모든 경로에 대해 인증 여부를 적용하는 예시입니다.
```java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().antMatchers("/**").authenticated();
    }
}
```

#### Session Management
`SessionManagementConfigurer`는 세션과 관련된 보안을 적용할 수 있도록 지원합니다. 예를 들어, 인증된 사용자의 세션이 생성되는 수를 제한하거나 [`Session Fixation Attack`](https://owasp.org/www-community/attacks/Session_fixation)을 방지할 수 있습니다.

다음은 사용자의 최대 세션 수를 제한하는 기능을 적용하는 예시입니다.
```java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.sessionManagement(session -> {
            session.maximumSessions(1);
            session.sessionConcurrency(concurrency -> {
                concurrency.maxSessionsPreventsLogin(true);
            });
        });
    }
}
```

#### Remember Me
`RememberMeConfigurer`는 로그인 유지 기능을 적용할 수 있도록 지원합니다.

다음은 메모리 기반의 로그인 유지 기능에 대한 예시입니다.
```java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.rememberMe()
            .userDetailsService(userDetailsService())
            .key("remember-me-key")
            .rememberMeCookieName("remember-me")
            .rememberMeParameter("remember-me")
            .tokenRepository(persistentTokenRepository())
            .tokenValiditySeconds((int) Duration.ofDays(1).toSeconds());
    }
    
    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        return new InMemoryTokenRepositoryImpl();
    }
}
```


#### Handling Logouts
보안 기능의 마무리는 애플리케이션에 인증된 주체에 대한 인증 정보를 제거하는 것입니다. 인증 정보를 제거하는 가장 기본적인 프로세스는 로그아웃을 수행하는 것입니다.

기본적으로 springSecurityFilterChain에 등록되는 `LogoutPageGeneratingWebFilter`는 로그아웃 페이지를 제공합니다.

```java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin(Customizer.withDefaults());
    }
}
```

### Configure WebSecurity

#### HttpFirewall
HttpFirewall은 잠재적 위험을 방지하는 기능을 제공합니다. `StrictHttpFirewall`는 기본적으로 적용되는 HttpFirewall 구현체로 잠재적인 위협이 있을만한 요청에 대한 URL 패턴이 있을 경우 요청이 거절될 수 있도록 합니다.

다음은 StrictHttpFirewall을 통해 HTTP 메소드를 제한하는 예시입니다.
```java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Bean
    public StrictHttpFirewall httpFirewall() {
        StrictHttpFirewall firewall = new StrictHttpFirewall();
        firewall.setAllowedHttpMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
        return firewall;
    }
    
    @Override
    public void configure(WebSecurity web) throws Exception {
        web.httpFirewall(httpFirewall());
    }
}
```

#### Ignoring Security
`IgnoredRequestConfigurer`는 스프링 시큐리티에 의해 보안이 적용되지 않도록 설정할 수 있는 기능을 제공합니다.

예를 들어, 다음과 같이 정적 리소스에 대한 보안을 수행하지 않을 수 있습니다.
```java
@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers("/images/**", "/css/**", "/js/**");
    }
}
```

---

스프링 프레임워크 기반의 웹 애플리케이션에 스프링 시큐리티 모듈을 통해 보안을 적용해보았습니다. 스프링 시큐리티는 이외에도 많은 보안 기능을 제공하지만 이 글을 통해 모두 다루어볼 수 없는 점 양해바랍니다.







