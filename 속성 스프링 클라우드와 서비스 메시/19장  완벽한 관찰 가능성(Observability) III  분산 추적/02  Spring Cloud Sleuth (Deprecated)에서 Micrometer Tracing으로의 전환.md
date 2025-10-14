## Spring Cloud Sleuth (Deprecated)에서 Micrometer Tracing으로의 전환

과거 스프링 클라우드 생태계에서 분산 추적을 구현하는 표준적인 방법은 **Spring Cloud Sleuth**였습니다. Sleuth는 `Trace ID`와 `Span ID`를 생성하고 전파하는 핵심 기능을 제공했으며, `Zipkin` 같은 추적 백엔드와 손쉽게 연동되었습니다.

하지만 19장 01절에서 설명했듯이, 업계의 표준이 \*\*OpenTelemetry(OTel)\*\*로 통일되면서, 스프링 팀은 Sleuth를 더 이상 사용하지 않고(Deprecated) OpenTelemetry 표준을 기반으로 하는 새로운 분산 추적 솔루션을 도입하기로 결정했습니다.

**Spring Boot 3.0 이상**을 사용하는 현대적인 프로젝트에서, 분산 추적을 위한 유일하고 올바른 선택은 바로 **Micrometer Tracing**입니다.

-----

### Micrometer Tracing: 관찰 가능성을 위한 통합 Facade

Micrometer는 "메트릭을 위한 SLF4J"라고 소개했습니다. 이제 그 역할이 '추적'까지 확장되었습니다. Micrometer는 이제 **메트릭과 추적을 모두 아우르는 '관찰 가능성 Facade'** 역할을 합니다.

**동작 방식:**

1.  **Micrometer Tracing API:** 개발자는 `Tracer`, `Span` 등 Micrometer가 제공하는 표준 추적 API를 사용하여 코드를 작성합니다.
2.  **브릿지 (Bridge):** 이 표준 API는 내부적으로 실제 추적 구현체와 연결됩니다. 현재 권장되는 표준 구현체는 단연 **OpenTelemetry**입니다.
3.  **자동 설정 (Auto-Configuration):** 스프링 부트는 이 모든 과정을 자동으로 설정하여, 개발자가 복잡한 설정 없이도 즉시 분산 추적을 사용할 수 있도록 지원합니다.

Sleuth에서 Micrometer Tracing으로의 전환은 단순히 라이브러리를 바꾸는 것을 넘어, 스프링 생태계가 독자적인 추적 방식을 버리고 업계 표준인 OpenTelemetry를 전면적으로 수용했음을 의미합니다.

-----

### Micrometer Tracing 적용하기 (Sleuth 마이그레이션)

#### 1\. 의존성 변경 (`build.gradle.kts`)

`spring-cloud-starter-sleuth` 의존성이 있다면 **반드시 제거**하고, Micrometer Tracing 관련 의존성을 추가합니다.

```kotlin
// build.gradle.kts
dependencies {
    // --- SLEUTH 의존성 제거 ---
    // implementation("org.springframework.cloud:spring-cloud-starter-sleuth")

    // --- MICROMETER TRACING 의존성 추가 ---
    // 1. Actuator는 기본적으로 필요
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    
    // 2. Micrometer Tracing과 OpenTelemetry를 연결하는 브릿지
    implementation("io.micrometer:micrometer-tracing-bridge-otel")

    // 3. 수집된 Trace를 Jaeger로 보내기 위한 OpenTelemetry Exporter
    implementation("io.opentelemetry:opentelemetry-exporter-jaeger")
    
    // (만약 Zipkin으로 보내고 싶다면 ...exporter-zipkin 사용)
}
```

#### 2\. `application.yml` 설정 변경

Sleuth에서 사용하던 `spring.sleuth.*` 설정 키는 모두 `management.tracing.*`으로 변경되었습니다.

```yaml
# application.yml
management:
  # 1. Tracing 기능 활성화 (기본값: true)
  tracing:
    sampling:
      # 2. 모든 요청을 추적하도록 샘플링 비율을 100%로 설정 (개발/테스트 환경용)
      #    프로덕션에서는 성능을 위해 10% (0.1) 등으로 조정
      probability: 1.0
  
  # 3. OpenTelemetry Exporter 설정 (Jaeger 예시)
  otlp:
    tracing:
      endpoint: http://localhost:14250 # Jaeger Collector의 gRPC 엔드포인트

# 4. 서비스 이름은 spring.application.name을 자동으로 사용
spring:
  application:
    name: order-service
```

#### 3\. 애플리케이션 코드 (수정 불필요\!)

만약 기존에 Sleuth의 **자동 계측** 기능에만 의존하고 있었다면, 애플리케이션 코드는 **단 한 줄도 수정할 필요가 없을 가능성이 높습니다.**

스프링 부트의 `RestTemplate`, `WebClient`, `@RestController`, `KafkaListener` 등에 대한 자동 계측 로직이 이제 Sleuth 대신 Micrometer Tracing에 의해 투명하게 처리됩니다. 17장에서 구현했던 MDC를 통한 `traceId` 로그 연동 또한 그대로 동작합니다.

-----

### 최고의 실무자를 위한 선택: OpenTelemetry Java 에이전트

위의 SDK 기반 설정은 매우 간편하지만, 프로덕션 환경에서 가장 권장되는 방식은 19장 01절에서 소개한 **OpenTelemetry Java 에이전트**를 사용하는 것입니다.

**적용 방식:**

1.  `build.gradle.kts`에는 `micrometer-tracing-bridge-otel`까지만 추가하고, `exporter-jaeger` 같은 특정 Exporter 의존성은 **제거**합니다.
2.  애플리케이션을 실행할 때 다음과 같이 **javaagent** 옵션을 추가합니다.
    ```bash
    java -javaagent:opentelemetry-javaagent.jar \
         -Dotel.service.name=order-service \
         -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
         -jar order-service.jar
    ```

**이 방식이 더 나은 이유:**

  * **더 넓은 계측 범위:** Java 에이전트는 스프링 부트의 자동 설정을 넘어서, JDBC 드라이버, Redis 클라이언트, gRPC 등 훨씬 더 광범위한 라이브러리를 코드 수정 없이 자동으로 계측합니다.
  * **일관성 및 중앙 제어:** 모든 서비스의 추적 방식을 에이전트 버전과 설정으로 중앙에서 일관되게 관리할 수 있습니다. 애플리케이션은 추적 로직에 대해 전혀 신경 쓸 필요가 없어집니다.

**결론적으로,** Sleuth에서 Micrometer Tracing으로의 전환은 OpenTelemetry라는 업계 표준에 합류하는 필수적인 과정입니다. SDK를 통한 간편한 설정도 가능하지만, 최고의 실무 환경에서는 **OpenTelemetry Java 에이전트**를 활용하여 애플리케이션 코드의 독립성과 관찰 가능성을 극대화하는 것이 가장 이상적인 접근 방식입니다.