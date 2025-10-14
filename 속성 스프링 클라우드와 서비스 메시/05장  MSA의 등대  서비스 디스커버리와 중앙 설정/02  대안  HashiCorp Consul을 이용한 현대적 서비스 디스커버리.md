## 대안: HashiCorp Consul을 이용한 현대적 서비스 디스커버리

Eureka를 통해 서비스 디스커버리의 기본 원리를 익혔으니, 이제 더 현대적이고 강력한 대안인 **HashiCorp Consul**을 살펴보겠습니다. Consul은 단순히 서비스 디스커버리 기능만 제공하는 것을 넘어, 분산 시스템에 필요한 여러 기능을 통합한 '서비스 네트워킹 플랫폼'입니다.

`Terraform`, `Vault`로 유명한 HashiCorp에서 개발한 Consul은 Go 언어로 작성되어 단일 바이너리로 실행되며, 클라우드 네이티브 환경에서 사실상의 표준 중 하나로 자리 잡고 있습니다.

-----

### Consul이 Eureka보다 나은 점

1.  **더 강력한 헬스 체크 (Health Checking):**

      * Eureka는 클라이언트가 보내는 '하트비트(Heartbeat)'에 의존합니다. 이는 "애플리케이션 프로세스가 살아있다"는 것만 알려줄 뿐, "DB 커넥션이 끊어지거나 외부 API 호출이 실패하는 등 **애플리케이션이 정말로 건강한(Healthy) 상태인가?**"는 알려주지 못합니다.
      * **Consul은** 주기적으로 각 서비스의 `/actuator/health` 같은 **HTTP API를 직접 호출**하거나 스크립트를 실행하여 **서비스의 '건강 상태'를 능동적으로 체크**합니다. 만약 헬스 체크가 실패하면, Consul은 즉시 해당 서비스 인스턴스를 '비정상(Unhealthy)'으로 표시하고 디스커버리 목록에서 제외합니다. 이는 훨씬 더 정교하고 신뢰성 높은 장애 감지 메커니즘입니다.

2.  **내장 Key/Value 저장소 (K/V Store):**

      * Consul은 분산 Key/Value 저장소를 내장하고 있습니다. 이를 활용하면 서비스 디스커버리뿐만 아니라, **분산 환경의 동적 설정 관리**(05장 03절), 기능 플래그(Feature Flag), 리더 선출(Leader Election) 등 다양한 용도로 활용할 수 있습니다.

3.  **멀티 데이터센터 지원:**

      * Consul은 처음부터 여러 데이터센터(리전)를 하나로 묶어 관리하는 기능을 염두에 두고 설계되었습니다. 대규모 글로벌 서비스에 훨씬 적합합니다.

4.  **서비스 메시 (Service Mesh) 지원:**

      * Consul Connect 기능을 통해 코드 변경 없이 서비스 간의 통신을 암호화(mTLS)하고, 세분화된 접근 제어를 수행하는 등 완전한 서비스 메시의 컨트롤 플레인(Control Plane) 역할을 할 수 있습니다. (21장 Istio와 유사)

-----

### Consul로 마이그레이션하기

Spring Cloud의 강력한 추상화 덕분에, Eureka에서 Consul로 전환하는 과정은 놀랍도록 간단합니다.

#### 1\. Consul 서버 실행 (Docker 활용)

개발 환경에서는 Docker를 이용해 Consul 서버를 실행하는 것이 가장 편리합니다.

```bash
# Consul 서버를 컨테이너로 실행
# 8500: HTTP API 및 UI 포트, 8600: DNS 포트
docker run -d -p 8500:8500 -p 8600:8600/udp --name=consul consul agent \
-server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0
```

이제 웹 브라우저에서 `http://localhost:8500`으로 접속하면 Consul의 관리 UI를 볼 수 있습니다.

#### 2\. 마이크로서비스 설정 변경 (`member-service` 기준)

`member-service`의 설정을 Eureka 클라이언트에서 Consul 클라이언트로 변경합니다.

**`build.gradle.kts` 수정:**

```kotlin
// member-service/build.gradle.kts
dependencies {
    // 1. Eureka 클라이언트 의존성 제거
    // implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")

    // 2. Consul 디스커버리 클라이언트 의존성 추가
    implementation("org.springframework.cloud:spring-cloud-starter-consul-discovery")

    // 3. (필수) Consul의 헬스 체크를 위한 Actuator 의존성 추가
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    
    // ...
}
```

**`application.yml` 수정:**

```yaml
# member-service/src/main/resources/application.yml

# spring.application.name: member-service (이 설정은 그대로 유지)

# --- 1. 기존 eureka 설정 블록 전체 삭제 ---
# eureka:
#   client:
#     ...

# --- 2. Consul 설정 블록 추가 ---
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        # 3. Consul이 이 경로를 주기적으로 호출하여 건강 상태를 체크
        health-check-path: /actuator/health
        # 4. 헬스 체크 주기
        health-check-interval: 15s
        # 5. 서비스 ID를 고유하게 만들어 동일 호스트에서 여러 인스턴스 실행 가능
        instance-id: ${spring.application.name}:${random.value}

# 6. Actuator의 health 엔드포인트 활성화
management:
  endpoints:
    web:
      exposure:
        include: health,info
```

`gateway-service`도 위와 동일하게 의존성과 `application.yml` 설정을 변경해 줍니다.

#### 3\. API 게이트웨이 설정 (변경 없음\!)

가장 놀라운 부분입니다. `gateway-service`의 라우팅 설정은 **단 한 글자도 수정할 필요가 없습니다.**

```yaml
# gateway-service/src/main/resources/application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: member-service-route
          # Spring Cloud의 LoadBalancerClient가 알아서 Consul을 바라보고 동작함
          uri: lb://member-service
          predicates:
            - Path=/api/v1/members/**
```

`lb://` 프로토콜은 Spring Cloud의 추상화 계층입니다. `classpath`에 Eureka 클라이언트가 있으면 Eureka로, Consul 클라이언트가 있으면 Consul로 자동 전환되어 동작합니다. 이것이 바로 프레임워크를 사용하는 강력한 이점입니다.

이제 `discovery-service`(Eureka) 대신 `consul` 컨테이너를 실행하고, 수정된 `member-service`와 `gateway-service`를 실행하면 모든 것이 이전과 동일하게, 하지만 훨씬 더 안정적인 헬스 체크 메커니즘 위에서 동작하는 것을 확인할 수 있습니다.