## Micrometer와 Spring Boot Actuator: JVM 및 비즈니스 메트릭 노출

프로메테우스(Prometheus)가 '인구 조사원'이라면, 우리 마이크로서비스는 조사원이 왔을 때 대답할 '자료'를 미리 준비하고 있어야 합니다. 즉, 자신의 상태(CPU, 메모리, API 응답 시간 등)를 프로메테우스가 이해할 수 있는 텍스트 형식으로 노출하는 HTTP 엔드포인트가 필요합니다.

이 역할을 **Spring Boot Actuator**와 **Micrometer**가 환상적인 조합으로 수행합니다.

  * **Spring Boot Actuator:** 16장에서 헬스 체크(`/actuator/health`)를 위해 이미 사용했던, 스프링 부트의 프로덕션 운영 기능 모듈입니다. Actuator는 애플리케이션의 내부 상태를 외부로 노출하는 다양한 엔드포인트를 제공하며, 그중에는 `/actuator/prometheus`도 포함됩니다.

  * **Micrometer:** \*\*"메트릭을 위한 SLF4J"\*\*라고 불리는 라이브러리입니다. `SLF4J`가 개발자가 `Logback`이나 `Log4j` 같은 실제 로깅 구현체를 직접 의존하지 않고 `Logger`라는 표준 인터페이스에 맞춰 코드를 작성하게 해주는 것처럼, `Micrometer`는 개발자가 `Prometheus`, `Datadog`, `New Relic` 같은 실제 모니터링 시스템을 직접 의존하지 않고 표준 API에 맞춰 메트릭 수집 코드를 작성하게 해줍니다.

애플리케이션은 Micrometer의 표준 API를 사용하여 메트릭을 기록하고, 우리는 `micrometer-registry-prometheus`라는 '어댑터' 의존성을 추가하는 것만으로 모든 메트릭이 프로메테우스 형식으로 자동 변환되어 노출됩니다.

-----

### 1\. 의존성 추가 및 Actuator 엔드포인트 노출

모든 마이크로서비스(`member-service`, `order-service` 등)의 `build.gradle.kts`에 다음 의존성을 추가합니다.

```kotlin
// build.gradle.kts
dependencies {
    // 1. Actuator 스타터
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    
    // 2. Micrometer의 프로메테우스 레지스트리 '어댑터'
    implementation("io.micrometer:micrometer-registry-prometheus")
    // ...
}
```

이제 `application.yml`에서 `/actuator/prometheus` 엔드포인트를 웹으로 노출시킵니다.

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        # health, info 외에 prometheus 엔드포인트를 추가로 노출
        include: health, info, prometheus
  metrics:
    tags: # 모든 메트릭에 공통 태그(레이블) 추가
      application: ${spring.application.name}
```

-----

### 2\. "공짜로 얻는" 자동 계측 메트릭

위의 의존성을 추가하고 설정을 마치는 것만으로도, 우리는 **단 한 줄의 코드도 작성하지 않고** 다음과 같은 엄청나게 유용한 메트릭들을 '공짜로' 얻게 됩니다.

  * **JVM 메트릭:**
      * `jvm_memory_used_bytes`: 힙 메모리 사용량
      * `jvm_gc_pause_seconds`: GC(Garbage Collection)로 인해 애플리케이션이 멈춘 시간
      * `jvm_threads_live_threads`: 현재 활성 스레드 수
  * **시스템 메트릭:**
      * `system_cpu_usage`: 시스템 전체 CPU 사용률
  * **웹 서버 메트릭 (Tomcat/Netty):**
      * `tomcat_sessions_active_current`: 현재 활성 세션 수
  * **Spring MVC / WebFlux 메트릭 (가장 유용\!):**
      * `http_server_requests_seconds_*`: **모든 HTTP API 엔드포인트**에 대해 자동으로 수집되는 지연 시간(Latency) 통계입니다. `uri`, `method`, `status` 코드가 **레이블**로 자동 추가되어 "`/api/v1/orders` POST 요청 중 `500` 에러가 발생한 요청 수"와 같은 정밀한 분석이 가능합니다.
  * **Kafka Consumer 메트릭:**
      * `kafka_consumer_fetch_manager_records_lag_max`: 컨슈머가 얼마나 뒤처져 있는지를 나타내는 '컨슈머 랙'. 이 지표가 계속 증가하면 시스템에 심각한 문제가 있다는 신호입니다.

서비스를 실행하고 웹 브라우저나 `curl`로 `http://localhost:8081/actuator/prometheus`에 접속하면, 위와 같은 수많은 메트릭이 프로메테우스 형식으로 출력되는 것을 확인할 수 있습니다.

-----

### 3\. 커스텀 비즈니스 메트릭 추가하기

Actuator가 제공하는 시스템 메트릭도 훌륭하지만, 모니터링의 진정한 힘은 **우리의 비즈니스와 직접 관련된 메트릭**을 추가할 때 발휘됩니다. 예를 들어, "주문이 얼마나 많이 생성되고 있는가?"를 측정해 봅시다.

`MeterRegistry`와 `Counter`를 사용하여 이 비즈니스 메트릭을 `OrderService`에 추가할 수 있습니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/service/OrderService.kt
import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import org.springframework.stereotype.Service

@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val meterRegistry: MeterRegistry // 1. MeterRegistry 주입
) {
    // 2. '주문 생성'을 위한 Counter 정의
    private val orderCounter: Counter = Counter
        .builder("orders.created.total") // 메트릭 이름
        .description("Total number of orders created")
        .register(meterRegistry) // 3. MeterRegistry에 등록

    fun createOrder(command: CreateOrderCommand): Long {
        // ... 주문 생성 비즈니스 로직 ...
        val savedOrder = orderRepository.save(newOrder)
        
        // 4. 주문이 성공적으로 생성되면 Counter의 값을 1 증가
        orderCounter.increment()

        return savedOrder.id!!
    }
}
```

이제 `/actuator/prometheus` 엔드포인트를 다시 확인하면, 우리가 직접 만든 `orders_created_total`이라는 메트릭이 새롭게 추가된 것을 볼 수 있습니다.

Micrometer와 Actuator를 통해, 우리는 시스템의 내부 상태와 비즈니스의 핵심 성과 지표(KPI)를 손쉽게 외부로 노출할 수 있습니다. 이제 남은 일은 '인구 조사원'인 프로메테우스가 이 정보를 주기적으로 수집하고, '관제 센터'인 Grafana가 이를 아름다운 대시보드로 보여주게 하는 것뿐입니다.