---
title: 스프링 웹 애플리케이션 - 이메일 발송
date: 2020-09-16
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
description: 잠만보와 함께하는 스프링 프레임워크 기반의 웹 애플리케이션
---

## Overview
본 글은 스프링 프레임워크 Version `5.2.8.RELEASE` 문서를 기반으로 작성하였습니다.

웹 애플리케이션에서 중요한 기능 중 하나는 이메일 발송 기능입니다. 예를 들어, 회원가입을 완료하였을 경우 안내 또는 추가적인 인증을 위하여 이메일을 발송할 수도 있으며 웹 애플리케이션에서 발생한 중요한 사항을 이메일로 제공해야할 필요성이 있습니다.

## Email
스프링 프레임워크는 [JavaMail](https://javaee.github.io/javamail/) 라이브러리를 사용하여 이메일을 발송할 수 있도록 지원합니다. 그리고 `JavaMailSender` 인터페이스를 통해 이메일에 대한 추상화를 지원합니다.

이메일 발송 기능을 추가하기 위하여 [`javax.mail`](https://mvnrepository.com/artifact/com.sun.mail/javax.mail/1.6.2)을 의존성에 추가합니다.
```groovy build.gradle
implementation 'com.sun.mail:javax.mail:1.6.2'
```

클래스패스에 mail.properties 파일을 생성하여 이메일을 보낼때 사용할 프로퍼티를 작성합니다. 기술한 메일 프로퍼티는 구글 이메일 SMTP 서버를 통해 메일을 발송하기 위한 정보입니다.
```properties mail.properties
mail.host=smtp.gmail.com
mail.port=587
mail.protocol=smtp
mail.default-encoding=UTF-8
mail.username=username
mail.password=password
mail.smtp.auth=true
mail.stmp.ssl.enable=true
mail.smtp.starttls.required=false
mail.smtp.starttls.enable=false
```

스프링 프레임워크는 기본적으로 `JavaMailSenderImpl`라는 JavaMailSender 구현체를 제공하므로 JavaMailSenderImpl를 `MailSender`로 등록합니다.
```java
@PropertySource({"classpath:/mail.properties"})
@Configuration
public class MailConfig implements EnvironmentAware {
    
    private static final String MAIL_HOST = "mail.host";
    private static final String MAIL_PORT = "mail.port";
    private static final String MAIL_PROTOCOL = "mail.protocol";
    private static final String MAIL_DEFAULT_ENCODING = "mail.default-encoding";
    private static final String MAIL_USERNAME = "mail.username";
    private static final String MAIL_PASSWORD = "mail.password";
    private static final String MAIL_SMTP_AUTH = "mail.smtp.auth";
    private static final String MAIL_SMTP_STARTTLS_REQUIRED = "mail.smtp.starttls.required";
    private static final String MAIL_SMTP_STARTTLS_ENABLE = "mail.smtp.starttls.enable";

    private Environment environment;

    public MailConfig() {}

    @Bean
    public MailSender mailSender() {
        String host = environment.getProperty(MAIL_HOST, String.class);
        int port = environment.getProperty(MAIL_PORT, Integer.class, -1);
        String protocol = environment.getProperty(MAIL_PROTOCOL, String.class);
        String defaultEncoding = environment.getProperty(MAIL_DEFAULT_ENCODING, String.class, StandardCharsets.UTF_8.displayName());
        String username = environment.getProperty(MAIL_USERNAME, String.class);
        String password = environment.getProperty(MAIL_PASSWORD, String.class);

        JavaMailSenderImpl javaMailSender = new JavaMailSenderImpl();
        javaMailSender.setHost(host);
        javaMailSender.setPort(port);
        javaMailSender.setProtocol(protocol);
        javaMailSender.setDefaultEncoding(defaultEncoding);
        javaMailSender.setUsername(username);
        javaMailSender.setPassword(password);

        boolean auth = environment.getProperty(MAIL_SMTP_AUTH, Boolean.class, false);
        boolean startTlsRequired = environment.getProperty(MAIL_SMTP_STARTTLS_REQUIRED, Boolean.class, false);
        boolean startTlsEnable = environment.getProperty(MAIL_SMTP_STARTTLS_ENABLE, Boolean.class, false);

        Properties javaMailProperties = new Properties();
        javaMailProperties.setProperty(MAIL_SMTP_AUTH, Boolean.toString(auth));
        javaMailProperties.setProperty(MAIL_SMTP_STARTTLS_ENABLE, Boolean.toString(startTlsEnable));
        javaMailProperties.setProperty(MAIL_SMTP_STARTTLS_REQUIRED, Boolean.toString(startTlsRequired));
        javaMailSender.setJavaMailProperties(javaMailProperties);
        return javaMailSender;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

}
```

이제 MailSender를 통해 이메일을 발송할 수 있습니다.
```java
@Service
public class MailService implements EnvironmentAware {

    private final MailSender mailSender;
    private Environment environment;

    public MailService(MailSender mailSender) {
        this.mailSender = mailSender;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
}
```

만약, 이메일 발송 시 `인증 오류`가 발생한다면 구글 계정에 대한 [`보안 수준이 낮은 앱의 액세스`](https://myaccount.google.com/lesssecureapps)를 활성화하시기 바랍니다. 구글 이메일 클라이언트로 SMTP 서버를 이용하기 위해서는 다음과 같은 정보로 사용해야 합니다.

- Host : smtp.gmail.com
- Auth required(mail.smtp.auth): 예
- SSL required(mail.smtp.ssl.enable): 예
- SSL Port: 465
- TLS/STARTTLS Port: 587
- TLS required: 예(사용 가능한 경우)


## Email Template Engine

> 업데이트 대기중입니다.

---

이제 여러분은 스프링 프레임워크 기반의 웹 애플리케이션에서 이메일을 발송할 수 있습니다.

