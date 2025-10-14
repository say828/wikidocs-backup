## Spring Cloud Gateway로 코틀린 기반 게이트웨이 구축

이제 우리 MSA의 '문지기'를 실제로 건설해 봅시다. 우리는 스프링 클라우드 생태계의 공식 차세대 API 게이트웨이인 **Spring Cloud Gateway**를 사용할 것입니다.

이전 세대 게이트웨이인 `Netflix Zuul 1`이 동기식(Blocking I/O) 서블릿 기반이었던 것과 달리, \*\*Spring Cloud Gateway는 Spring WebFlux와 Netty 기반의 완전한 비동기/논블로킹(Async/Non-blocking)\*\*으로 설계되었습니다. 이는 적은 리소스로 대규모 트래픽을 효율적으로 처리해야 하는 게이트웨이의 역할에 완벽하게 부합하며, 12장에서 다룰 코루틴 기반 MSA로의 전환과도 아키텍처적 일관성을 가집니다.

-----

### 1\. 새로운 'gateway-service' 프로젝트 생성

API 게이트웨이는 기존 `member-service`와는 완전히 분리된 **별개의 스프링 부트 프로젝트**입니다. `gateway-service`라는 이름으로 새로운 프로젝트를 생성합니다.

#### `build.gradle.kts` 의존성 설정

게이트웨이 프로젝트의 `build.gradle.kts` 파일에 핵심 의존성을 추가합니다.

```kotlin
// gateway-service/build.gradle.kts
plugins {
    id("org.springframework.boot") version "3.2.5"
    id("io.spring.dependency-management") version "1.1.5"
    kotlin("jvm") version "1.9.24"
    kotlin("plugin.spring") version "1.9.24"
}

// 1. 스프링 클라우드 의존성 버전을 관리하기 위한 BOM(Bill of Materials) 설정
dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.1")
    }
}

dependencies {
    // 2. Spring Cloud Gateway 핵심 스타터
    // (내부에 spring-boot-starter-webflux, netty 포함)
    implementation("org.springframework.cloud:spring-cloud-starter-gateway")
    
    // 3. 코루틴과 Reactor(WebFlux)를 함께 사용하기 위한 의존성 (향후 사용)
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")

    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlin:kotlin-stdlib")

    // (인증/인가를 위한 의존성은 04.03절에서 추가)

    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

  * **주의:** `spring-cloud-starter-gateway`는 내부에 `spring-boot-starter-webflux`를 포함하므로, 동기식 `spring-boot-starter-web` (Tomcat 기반) 의존성을 **절대 함께 추가하면 안 됩니다.** 두 의존성이 충돌합니다.

-----

### 2\. 라우팅(Routing) 규칙 설정

게이트웨이의 핵심 기능은 '라우팅'입니다. 이 규칙은 `application.yml` 파일에 선언적으로 정의합니다.

```yaml
# gateway-service/src/main/resources/application.yml

server:
  port: 8080 # 1. 게이트웨이는 MSA의 대표 포트인 8080번을 사용

spring:
  application:
    name: gateway-service # 2. 서비스 이름 정의

  cloud:
    gateway:
      # 3. 라우팅 규칙 정의 (routes는 List 형태)
      routes:
        # --- 회원 서비스 라우트 ---
        - id: member-service-route # 4. 각 라우트의 고유 ID
          
          # 5. 이 라우트가 최종적으로 요청을 전달할 목적지 URI
          # (주: 지금은 하드코딩. 05장에서 서비스 디스커버리를 통해 'lb://MEMBER-SERVICE'로 변경 예정)
          uri: http://localhost:8081
          
          # 6. 이 라우트가 적용될 조건 (Predicates)
          predicates:
            - Path=/api/v1/members/** # 7. /api/v1/members/로 시작하는 모든 경로

        # --- (미래) 상품 서비스 라우트 ---
        - id: product-service-route
          uri: http://localhost:8082 # (아직 존재하지 않는 서비스)
          predicates:
            - Path=/api/v1/products/**

        # --- (미래) 주문 서비스 라우트 ---
        - id: order-service-route
          uri: http://localhost:8083 # (아직 존재하지 않는 서비스)
          predicates:
            - Path=/api/v1/orders/**
```

-----

### 3\. 게이트웨이 애플리케이션 실행

메인 애플리케이션 클래스는 일반적인 스프링 부트 애플리케이션과 동일합니다.

```kotlin
// gateway-service/src/main/kotlin/com/ecommerce/gateway/GatewayApplication.kt
package com.ecommerce.gateway

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class GatewayApplication

fun main(args: Array<String>) {
    runApplication<GatewayApplication>(*args)
}
```

### 동작 방식 확인

이제 2개의 서비스를 실행합니다.

1.  `member-service`를 `8081` 포트로 실행합니다.
2.  `gateway-service`를 `8080` 포트로 실행합니다.

그리고 Postman이나 curl을 사용하여 \*\*게이트웨이(8080 포트)\*\*로 API를 호출합니다.

```bash
# 회원 가입 요청 (게이트웨이를 통해)
curl -X POST http://localhost:8080/api/v1/members/join \
     -H "Content-Type: application/json" \
     -d '{"email":"gateway@email.com", "name":"Gateway User", "address":"Seoul"}'

# 회원 조회 요청 (게이트웨이를 통해)
curl http://localhost:8080/api/v1/members/1
```

**흐름:**

1.  클라이언트의 요청이 `localhost:8080` (게이트웨이)에 도착합니다.
2.  게이트웨이는 요청 경로가 `/api/v1/members/1`이므로 `Path` 조건자(Predicate)에 따라 `member-service-route`가 일치함을 확인합니다.
3.  게이트웨이는 이 요청을 `uri`에 명시된 `http://localhost:8081`로 그대로 전달(Proxy)합니다.
4.  `member-service`는 `localhost:8081/api/v1/members/1`로 요청을 받은 것처럼 처리하고 응답을 반환합니다.
5.  게이트웨이는 `member-service`의 응답을 받아 다시 클라이언트에게 전달합니다.

-----

이것으로 우리 MSA의 '정문'과 '안내판(라우팅 규칙)'이 완성되었습니다. 클라이언트는 이제 내부 서비스들의 복잡한 주소를 전혀 몰라도 됩니다.

다음 단계는 이 정문에 '보안 요원'을 배치하는 것입니다. 즉, 모든 요청에 공통적으로 적용될 로직(필터)을 구현해 보겠습니다.