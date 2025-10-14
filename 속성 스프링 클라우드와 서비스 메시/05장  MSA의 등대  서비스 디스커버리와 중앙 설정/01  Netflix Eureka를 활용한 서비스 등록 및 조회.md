## Netflix Eureka를 활용한 서비스 등록 및 조회

05장 00절에서 서비스 디스커버리의 '원리'를 배웠으니, 이제 실제 '구현체'를 다룰 차례입니다. 우리는 스프링 클라우드 생태계에서 가장 오랫동안 사랑받아 온 **Netflix Eureka**를 사용해 보겠습니다.

Eureka 아키텍처는 두 가지 핵심 컴포넌트로 구성됩니다.

  * **Eureka Server:** 모든 마이크로서비스가 자신의 정보를 등록하는 '서비스 레지스트리' 서버입니다. (우리의 "전화번호부")
  * **Eureka Client:** 각 마이크로서비스(`member-service`, `gateway-service` 등)에 포함되는 라이브러리입니다. Eureka Server에 자신을 등록하고, 다른 서비스의 정보를 조회하는 역할을 합니다.

-----

### 1\. Eureka Server (discovery-service) 구축하기

먼저, '전화번호부' 역할을 할 Eureka 서버를 **별개의 스프링 부트 프로젝트**로 만듭니다.

#### 1-1. `discovery-service` 프로젝트 생성 및 의존성 추가

`discovery-service`라는 이름으로 새 프로젝트를 생성하고, `build.gradle.kts`에 Eureka 서버 의존성을 추가합니다.

```kotlin
// discovery-service/build.gradle.kts
plugins { /* ... */ }

// Spring Cloud BOM 설정 필수
dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.1")
    }
}

dependencies {
    // 1. Eureka 서버 스타터
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-server")
    // ...
}
```

#### 1-2. Eureka Server 활성화 및 설정

메인 애플리케이션 클래스에 `@EnableEurekaServer` 어노테이션을 추가하여 Eureka 서버 기능을 활성화합니다.

```kotlin
// discovery-service/src/main/kotlin/com/ecommerce/discovery/DiscoveryApplication.kt
package com.ecommerce.discovery

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer

@SpringBootApplication
@EnableEurekaServer // Eureka 서버 기능을 활성화
class DiscoveryApplication

fun main(args: Array<String>) {
    runApplication<DiscoveryApplication>(*args)
}
```

`application.yml` 파일에서 Eureka 서버의 동작을 설정합니다.

```yaml
# discovery-service/src/main/resources/application.yml
server:
  port: 8761 # 1. Eureka 서버의 기본 포트

spring:
  application:
    name: discovery-service

eureka:
  client:
    # 2. Eureka 서버는 자신을 '클라이언트'로서 등록하지 않음
    register-with-eureka: false
    # 3. 레지스트리 정보를 로컬에 캐싱하지 않음
    fetch-registry: false
  server:
    # (개발 환경) 클라이언트의 하트비트가 없어도 즉시 제거하지 않도록 설정
    eviction-interval-timer-in-ms: 10000 
```

이제 `discovery-service`를 실행하고 웹 브라우저에서 `http://localhost:8761`로 접속하면, Eureka 대시보드를 볼 수 있습니다. 현재는 "Instances currently registered with Eureka" 항목이 비어있을 것입니다.

-----

### 2\. 마이크로서비스를 Eureka Client로 등록하기

이제 기존 `member-service`와 `gateway-service`가 Eureka 서버에 자신을 등록하도록 설정합니다.

#### 2-1. Eureka Client 의존성 추가

`member-service`와 `gateway-service`의 `build.gradle.kts`에 Eureka 클라이언트 의존성을 추가합니다.

```kotlin
// member-service/build.gradle.kts & gateway-service/build.gradle.kts
dependencies {
    // 1. Eureka 클라이언트 스타터
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
    // ... 기존 의존성 ...
}
```

#### 2-2. `application.yml` 설정

각 서비스의 `application.yml` 파일에 Eureka 서버 주소와 자신의 '서비스 이름'을 알려줍니다.

```yaml
# member-service/src/main/resources/application.yml
spring:
  application:
    name: member-service # 1. Eureka에 등록될 '서비스 이름'. 대문자를 권장 (MEMBER-SERVICE)

eureka:
  client:
    service-url:
      # 2. Eureka 서버(들)의 주소.
      defaultZone: http://localhost:8761/eureka/
  instance:
    # 3. IP 주소로 등록되도록 설정
    prefer-ip-address: true
```

`gateway-service`의 `application.yml`에도 위와 동일한 `eureka.client` 설정을 추가하고, `spring.application.name`을 `gateway-service`로 설정합니다.

-----

### 3\. API 게이트웨이 라우팅 동적으로 변경하기

마지막으로, 04장에서 하드코딩했던 `gateway-service`의 라우팅 설정을 Eureka와 연동되도록 수정합니다.

```yaml
# gateway-service/src/main/resources/application.yml
spring:
  application:
    name: gateway-service
  cloud:
    gateway:
      # 1. 서비스 디스커버리 기능 활성화
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true # 서비스 ID를 소문자로 사용
      routes:
        - id: member-service-route
          # 2. 'lb'는 Load Balancer를 의미
          # "Eureka에서 'member-service'라는 이름의 서비스 목록을 찾아
          # 로드 밸런싱하여 요청을 보내라"
          uri: lb://member-service
          predicates:
            - Path=/api/v1/members/**
        # ... 다른 서비스 라우트 ...
```

  * **`lb://MEMBER-SERVICE`**: Spring Cloud Gateway는 `lb` 프로토콜을 보면, Eureka 같은 서비스 레지스트리에게 `MEMBER-SERVICE`라는 이름의 인스턴스 목록을 물어봅니다. 그리고 목록을 받아 내부적으로 **클라이언트 사이드 로드 밸런싱**(기본값: Round-Robin)을 수행하여 요청을 전달합니다.

-----

### 동작 확인

1.  `discovery-service` (포트 8761)를 실행합니다.
2.  `member-service` (포트 8081)를 실행합니다.
3.  웹 브라우저에서 `http://localhost:8761`에 접속하여 Eureka 대시보드에 `MEMBER-SERVICE`가 등록되었는지 확인합니다.
4.  `gateway-service` (포트 8080)를 실행합니다.
5.  Postman 등으로 `http://localhost:8080/api/v1/members/1` (게이트웨이 주소)로 요청을 보냅니다.

이제 요청은 `게이트웨이` -\> `Eureka 서버` (주소 조회) -\> `member-service` (실제 호출)의 흐름으로 동적으로 처리됩니다. `member-service`의 IP나 포트가 변경되거나 여러 대로 늘어나도, 게이트웨이의 설정은 더 이상 변경할 필요가 없습니다. 이것이 바로 서비스 디스커버리의 힘입니다.