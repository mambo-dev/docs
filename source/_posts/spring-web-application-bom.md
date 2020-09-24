---
title: 스프링 웹 애플리케이션 - BOM(Bill of Materials)
date: 2020-09-24
categories:
- 스프링 프레임워크 가이드
tags:
- 스프링 프레임워크
- 스프링 튜토리얼
---

## Overview
스프링 부트 프로젝트에서는 의존성을 추가할 때 버전을 명시하지 않습니다. 이번 글은 스프링 프레임워크에서 제공하는 의존성 관리 기능에 대해서 알아보고 의존성 버전 관리를 위임하는 방법에 대해서 배우도록 하겠습니다.

## Dependency Management Gradle Plugin
우리는 스프링 프레임워크 기반의 웹 애플리케이션에서 사용되는 의존성을 다음과 같이 추가하였습니다.

```groovy build.gradle
dependencies {
    implementation 'org.springframework:spring-webmvc:5.2.8.RELEASE'

    implementation 'javax.servlet:javax.servlet-api:4.0.1'

    implementation 'org.apache.tomcat.embed:tomcat-embed-core:9.0.37'
    implementation 'org.apache.tomcat.embed:tomcat-embed-jasper:9.0.37'

    implementation 'org.slf4j:slf4j-api:1.7.30'
    implementation 'ch.qos.logback:logback-classic:1.2.3'
    implementation 'ch.qos.logback:logback-access:1.2.3'

    implementation 'org.freemarker:freemarker:2.3.30'
    implementation 'org.springframework:spring-context-support:5.2.8.RELEASE'

    implementation 'com.sun.mail:javax.mail:1.6.2'

    testCompile 'org.springframework:spring-test:5.2.8.RELEASE'
    testCompile group: 'junit', name: 'junit', version: '4.12'
}
```

스프링 프레임워크에서 제공하는 `spring-webmvc`와 `spring-context-support` 그리고 `spring-test`와 같이 5.2.8.RELEASE 버전을 동일하게 지정하였습니다. 이렇게 연관된 의존성에 대한 버전을 별도로 명시하는 것이 번거롭다고 느낄 수 있습니다.

### io.spring.dependency-management
스프링에서는 의존성 버전 관리를 위해 [`io.spring.dependency-management`](https://plugins.gradle.org/plugin/io.spring.dependency-management) 플러그인을 제공합니다. `io.spring.dependency-management`는 dependencyManagement 구문을 통해 의존성 관리를 위한 기능을 제공합니다. 그리고 imports를 통해 Maven BOM을 적용할 수 있습니다.
```groovy
plugins {
    id "io.spring.dependency-management" version "1.0.10.RELEASE"
}

dependencyManagement {
    imports {
        mavenBom 'org.springframework:spring-framework-bom:5.2.8.RELEASE'
    }
}
```

## Bill of Materials
`Maven BOM(Bill of Materials)`을 활용하면 프로젝트에서 사용되는 의존성을 쉽게 관리할 수 있습니다. 그리고 앞서 `io.spring.dependency-management` 플러그인을 추가하였으므로 BOM을 적용할 수 있게되었습니다. 

그리고 위에서 추가한 `spring-framework-bom`과 같이 스프링 프레임워크에서 제공하는 여러가지 BOM이 있습니다.

- [Spring Framework (Bill of Materials)](https://mvnrepository.com/artifact/org.springframework/spring-framework-bom)
- [Spring Session Maven Bill of Materials (BOM)](https://mvnrepository.com/artifact/org.springframework.session/spring-session-bom)
- [Spring Integration (Bill of Materials)](https://mvnrepository.com/artifact/org.springframework.integration/spring-integration-bom)
- [Spring Data Release Train BOM](https://mvnrepository.com/artifact/org.springframework.data/spring-data-releasetrain)
- [Spring Security BOM](https://mvnrepository.com/artifact/org.springframework.security/spring-security-bom)
- [Spring Boot Dependencies](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-dependencies)

스프링 프레임워크 기반의 웹 애플리케이션에서 사용되는 의존성을 별도로 지정하지 않아도 BOM에 의해 의존성에 대한 버전이 관리됩니다.

### spring-framework-bom
`spring-framework-bom`은 스프링 프레임워크에 대한 기본 모듈인 `spring-core`와 `spring-wemvc`와 같은 의존성들을 관리합니다. 따라서, 다음과 같이 기본 모듈에 대한 버전을 제거할 수 있습니다.

```groovy build.gradle
plugins {
    id 'java'
    id "io.spring.dependency-management" version "1.0.10.RELEASE"
}

dependencyManagement {
    imports {
        mavenBom 'org.springframework:spring-framework-bom:5.2.8.RELEASE'
    }
}

dependencies {
    implementation 'org.springframework:spring-webmvc'
    implementation 'org.springframework:spring-context-support'
    implementation 'org.springframework:spring-jdbc'
    implementation 'org.springframework:spring-webflux'
    implementation 'org.springframework:spring-tx'
    testCompile 'org.springframework:spring-test'
}
```

### spring-session-bom
`spring-session-bom`은 스프링 세션 프로젝트에 대한 모듈을 관리합니다. 따라서, `spring-session-core`와 `spring-session-data-redis`에 대한 의존성 버전을 제거할 수 있게 됩니다.

```groovy build.gradle
plugins {
    id 'java'
    id "io.spring.dependency-management" version "1.0.10.RELEASE"
}

dependencyManagement {
    imports {
        mavenBom 'org.springframework.session:spring-session-bom:Dragonfruit-RELEASE'
    }
}

dependencies {
    implementation 'org.springframework.session:spring-session-core'
    implementation 'org.springframework.session:spring-session-data-redis'
}
```


### spring-security-bom
`spring-security-bom`은 스프링 시큐리티 프로젝트에 대한 모듈을 관리합니다. 따라서, `spring-security-core`와 `spring-security-config`에 대한 의존성 버전을 제거할 수 있게 됩니다.

```groovy build.gradle
plugins {
    id 'java'
    id "io.spring.dependency-management" version "1.0.10.RELEASE"
}

dependencyManagement {
    imports {
        mavenBom 'org.springframework.security:spring-security-bom:5.3.4.RELEASE'
    }
}

dependencies {
    implementation 'org.springframework.security:spring-security-web:5.3.4.RELEASE'
    implementation 'org.springframework.security:spring-security-config:5.3.4.RELEASE'
}
```

### spring-boot-dependencies
`spring-boot-dependencies`은 스프링 부트 프로젝트에서 사용되는 의존성들을 관리하는 BOM입니다. 이 BOM에서는 `javax.servlet-api`와 `tomcat-embed-core`와 같은 스프링 프레임워크와 함께 사용되는 여러가지 의존성에 대한 버전도 같이 관리됩니다. 그리고 `spring-framework-bom`와 `spring-session-bom`도 포함됩니다.

따라서, 다음과 같이 스프링 프레임워크에서 제공하는 모듈이 아니어도 의존성 버전을 관리할 수 있게 됩니다.
```groovy build.gradle
plugins {
    id 'java'
    id "io.spring.dependency-management" version "1.0.10.RELEASE"
}

dependencyManagement {
    imports {
        mavenBom 'org.springframework.boot:spring-boot-dependencies:2.3.3.RELEASE'
    }
}

dependencies {
    implementation 'org.springframework:spring-webmvc'

    implementation 'javax.servlet:javax.servlet-api'

    implementation 'org.apache.tomcat.embed:tomcat-embed-core'
    implementation 'org.apache.tomcat.embed:tomcat-embed-jasper'

    implementation 'org.slf4j:slf4j-api'
    implementation 'ch.qos.logback:logback-classic'
    implementation 'ch.qos.logback:logback-access'

    implementation 'org.freemarker:freemarker'
    implementation 'org.springframework:spring-context-support'

    implementation 'com.sun.mail:javax.mail:1.6.2'
    
    implementation 'com.zaxxer:HikariCP'
    implementation 'org.springframework:spring-jdbc'
    implementation 'org.postgresql:postgresql'

    implementation 'org.springframework:spring-webflux'
    implementation 'io.projectreactor.netty:reactor-netty'

    implementation 'org.springframework.session:spring-session-core'
    implementation 'org.springframework.session:spring-session-data-redis'
    implementation 'io.lettuce:lettuce-core'

    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'com.google.code.gson:gson'
    implementation 'net.minidev:json-smart'

    implementation 'org.quartz-scheduler:quartz'
    implementation 'org.quartz-scheduler:quartz-jobs'
    implementation 'org.springframework:spring-tx'

    testCompile 'org.springframework:spring-test'
    testCompile group: 'junit', name: 'junit'
}
```

`javax.mail`와 같이 BOM에 의해 관리되는 의존성이 아니라면 별도로 버전을 명시해야합니다.

## Overriding Versions in a BOM
때때로, 필요에 의해 BOM에 의해 관리되는 의존성 버전을 변경해야할 필요성이 있습니다. 물론, 의존성을 추가할 때 별도로 지정할 수 있지만 `io.spring.dependency-management`에서 제공하는 방법을 활용하는 것이 좋습니다.

### bomProperty
```groovy build.gradle
dependencyManagement {
    imports {
        mavenBom('org.springframework.boot:spring-boot-dependencies:2.3.3.RELEASE') {
            bomProperty 'tomcat.version', '9.0.38'
        }
    }
}
```

### ext
```groovy build.gradle
ext['tomcat.version'] = '9.0.38'
ext['postgresql.version'] = '42.2.14'
```

### gradle.properties
```properties gradle.properties
tomcat.version=9.0.38
postgresql.version=42.2.14
```

---

스프링 프레임워크에서 제공하는 그레이들 플러그인과 BOM을 적용하여 의존성 버전 관리를 위임하였습니다. 이렇게 의존성 버전 관리를 위임하는 것은 스프링 프레임워크 기반의 애플리케이션을 만들때 사용되는 여러가지 의존성에 대한 버전에 대한 고민을 해결해주는 장점이 있습니다. 스프링 프레임워크에서 제공하는 BOM을 이용하므로써 스프링 프레임워크 모듈과 가장 적합한 호환성을 제공하는 버전을 적용할 수 있게 됩니다.





