## 분산 환경의 설정 관리: Spring Cloud Config Server 구축

05장 02절에서 우리는 Consul의 K/V 저장소를 통해 '동적 설정 관리'가 가능하다고 언급했습니다. 이제 이 '중앙 설정 관리'라는 주제를 본격적으로 다뤄보겠습니다.

MSA가 5개, 10개, 100개로 늘어남에 따라, 우리는 새로운 '설정 지옥'에 빠지게 됩니다.

  * **설정의 중복:** Consul 서버 주소, Kafka 브로커 주소, DB 접속 정보 등 공통된 설정이 100개의 `application.yml` 파일에 복사-붙여넣기 됩니다.
  * **변경의 어려움:** 개발 DB의 비밀번호가 변경되면, 개발자 10명이 각자 맡은 10개의 서비스 저장소(Repository)에 들어가 `application.yml` 파일을 수정하고, 커밋하고, 푸시해야 합니다. 실수가 발생할 수밖에 없습니다.
  * **보안의 취약성:** DB 비밀번호나 JWT 비밀 키 같은 민감 정보가 각 서비스의 Git 저장소에 그대로 노출됩니다.
  * **동적 반영의 부재:** 타임아웃 값을 `5s`에서 `7s`로 바꾸는 간단한 작업조차, 서비스를 재빌드하고 재배포해야만 적용됩니다.

이 문제를 해결하기 위해 스프링 클라우드 생태계는 **Spring Cloud Config**라는 강력한 중앙 설정 관리 솔루션을 제공합니다.

-----

### Spring Cloud Config Server 아키텍처

1.  **설정 정보 Git 저장소 (Config Repository):**

      * 모든 `application.yml` 파일들을 애플리케이션 코드와 분리하여, **별도의 Git 저장소**에서 중앙 관리합니다. (이것이 핵심입니다\!)
      * 설정 변경 이력이 Git에 모두 남으므로, 버전 관리 및 감사(Audit)가 용이해집니다.

2.  **Config Server:**

      * 설정 Git 저장소와 연결되어 있는 독립적인 스프링 부트 애플리케이션입니다.
      * 다른 마이크로서비스로부터 설정 정보 요청을 받으면, Git 저장소에서 적절한 `.yml` 파일을 읽어 HTTP를 통해 전달해 줍니다.

3.  **Config Client (in Microservices):**

      * 각 마이크로서비스(`member-service` 등)는 `spring-cloud-starter-config` 의존성을 가집니다.
      * 서비스가 시작될 때, 자신의 `application.name`과 `profile` 정보를 가지고 Config Server에 "내 설정 파일 좀 줘\!"라고 요청합니다.
      * Config Server로부터 설정을 받아와, 마치 자신의 `application.yml` 파일인 것처럼 메모리에 로드한 뒤 애플리케이션을 부트스트랩합니다.

-----

### 1\. 설정 정보 Git 저장소 준비

먼저, 로컬 또는 GitHub에 설정을 위한 별도의 Git 저장소를 생성합니다. 그리고 다음과 같은 파일들을 만듭니다.

`https://github.com/my-org/ecommerce-config-repo` (예시)

#### `application.yml` (모든 서비스 공통 설정)

```yaml
# 모든 서비스에 공통으로 적용될 설정
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

#### `member-service.yml` (`member-service` 전용 설정)

```yaml
# member-service의 모든 프로파일에 적용될 설정
server:
  port: 8081
management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh # 'refresh' 엔드포인트 노출 (05.04절)
```

#### `gateway-service.yml` (`gateway-service` 전용 설정)

```yaml
server:
  port: 8080
spring:
  cloud:
    gateway:
      # ... 라우팅 규칙 ...
```

-----

### 2\. Config Server (config-service) 구축하기

'Config Server' 역할을 할 **별개의 스프링 부트 프로젝트**를 생성합니다.

#### `config-service` 의존성 추가 및 활성화

```kotlin
// config-service/build.gradle.kts
dependencies {
    // 1. Config Server 스타터
    implementation("org.springframework.cloud:spring-cloud-config-server")
    // 2. Config Server도 Eureka에 등록하여 고가용성 확보
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
}
```

```kotlin
// config-service/src/main/kotlin/com/ecommerce/config/ConfigApplication.kt
package com.ecommerce.config

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.cloud.config.server.EnableConfigServer

@SpringBootApplication
@EnableConfigServer // Config Server 기능 활성화
class ConfigApplication

fun main(args: Array<String>) {
    runApplication<ConfigApplication>(*args)
}
```

#### `config-service`의 `application.yml` 설정

```yaml
# config-service/src/main/resources/application.yml
server:
  port: 8888 # Config Server의 기본 포트

spring:
  application:
    name: config-service
  cloud:
    config:
      server:
        git:
          # 1. 설정 Git 저장소 주소
          uri: https://github.com/my-org/ecommerce-config-repo
          # (Private 저장소일 경우 username, password 등 추가)
          default-label: main # 기본 브랜치
```

-----

### 3\. 마이크로서비스를 Config Client로 설정하기

이제 `member-service`가 자신의 로컬 `application.yml`이 아닌, `config-service`로부터 설정을 받아오도록 변경합니다.

#### `member-service` 의존성 추가 및 설정 파일 변경

`member-service`의 `build.gradle.kts`에 Config Client 의존성을 추가합니다.

```kotlin
// member-service/build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-config")
    // ...
}
```

`member-service`의 **`src/main/resources/application.yml`** 파일을 **삭제**하고, 대신 **`bootstrap.yml`** 파일을 생성합니다. (또는 최신 스프링 부트 방식인 `spring.config.import` 사용)

> **왜 `bootstrap.yml`인가?**
> `application.yml`은 애플리케이션의 '일반 설정' 파일입니다. 하지만 "Config Server의 주소가 어디인가?"라는 설정은, `application.yml`을 읽어오기 **전에** 먼저 알아야 하는 '최우선 설정'입니다. `bootstrap.yml`은 `application.yml`보다 먼저 로드되는 '부트스트랩 컨텍스트' 설정 파일입니다.

```yaml
# member-service/src/main/resources/bootstrap.yml
spring:
  application:
    name: member-service # 1. Config Server에 어떤 파일을 요청할지 식별하는 ID
  cloud:
    config:
      # 2. Config Server의 주소
      uri: http://localhost:8888
      # (만약 Config Server가 여러 대라면 fail-fast, retry 등 설정 가능)
```

이제 `member-service`의 기존 `application.yml`에 있던 `server.port=8081` 같은 설정은 모두 **삭제**하고, `bootstrap.yml`에 명시된 최소한의 설정만 남깁니다.

-----

### 동작 확인

1.  `config-repo` Git 저장소에 설정 파일들을 푸시합니다.
2.  `discovery-service`(Eureka)를 실행합니다.
3.  `config-service`(포트 8888)를 실행합니다.
4.  `member-service`를 실행합니다.

`member-service`의 시작 로그를 보면, 로컬에 `server.port` 설정이 없음에도 불구하고 `8081` 포트로 정상적으로 실행되는 것을 확인할 수 있습니다. 이는 `member-service`가 시작 시 `config-service`에 접속하여 Git 저장소에 있는 `member-service.yml` 파일의 설정을 성공적으로 가져왔다는 의미입니다.

이제 모든 설정이 중앙 Git 저장소에서 관리됩니다. DB 주소를 변경하고 싶다면, Git 저장소의 파일 하나만 수정하고 커밋하면 됩니다. 하지만 아직 한 가지 문제가 남았습니다. 설정을 변경해도, 이미 실행 중인 서비스들은 이를 알지 못합니다. 다음 절에서는 이 문제를 해결해 보겠습니다.