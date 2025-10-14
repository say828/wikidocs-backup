## Spring Cloud Stream: 추상화를 통한 유연한 메시징 구현

07장 02절에서 우리는 메시지 브로커로 `Kafka`를 선택했습니다. 그렇다면 이제 `order-service` 코드에 `spring-kafka` 라이브러리를 사용하여 직접 Kafka 프로듀서 코드를 작성해야 할까요?

```kotlin
// (나쁜 예시) Kafka에 직접 의존하는 코드 👎
@Service
class OrderService(private val kafkaTemplate: KafkaTemplate<String, OrderPaidEvent>) {
    fun someMethod() {
        val event = OrderPaidEvent(...)
        // 'ecommerce.orders.paid'라는 Kafka 토픽 이름을 코드에 직접 사용
        kafkaTemplate.send("ecommerce.orders.paid", event)
    }
}
```

이 코드는 동작하지만, 우리의 **비즈니스 로직이 `Kafka`라는 특정 기술에 강하게 결합**되어 버렸습니다. 만약 1년 뒤, RabbitMQ가 더 적합한 시나리오가 생겨 브로커를 교체해야 한다면, 이 모든 메시징 관련 코드를 전부 재작성해야 합니다.

이러한 문제를 해결하기 위해, 우리는 **Spring Cloud Stream**이라는 강력한 추상화 프레임워크를 사용합니다.

-----

### JPA와 같은 추상화의 힘

**Spring Cloud Stream**은 `Kafka`, `RabbitMQ`, `AWS SQS`, `Google Pub/Sub` 등 다양한 메시지 브로커 위에서 동작하는 **추상화 계층**입니다.

  * **JPA가 하는 일:** 개발자는 `JpaRepository`라는 '표준 인터페이스'에 맞춰 코드를 작성합니다. 그러면 `application.yml`의 설정만 바꾸어 DB를 MySQL에서 PostgreSQL로 교체할 수 있습니다.
  * **Spring Cloud Stream이 하는 일:** 개발자는 `Function`, `Consumer`, `Supplier`라는 '표준 함수'에 맞춰 비즈니스 로직을 작성합니다. 그러면 `build.gradle.kts`의 의존성과 `application.yml`의 설정만 바꾸어 메시지 브로커를 Kafka에서 RabbitMQ로 교체할 수 있습니다.

이 추상화 덕분에, 우리의 **핵심 비즈니스 로직은 특정 메시징 기술로부터 완벽하게 분리**되어 테스트 용이성과 기술 유연성을 확보하게 됩니다.

-----

### Spring Cloud Stream 3.x의 핵심 개념 (함수형 모델)

최신 Spring Cloud Stream은 `Bean`으로 등록된 \*\*함수(Function)\*\*를 기반으로 동작합니다.

1.  **바인더 (Binder):** 특정 메시지 브로커와의 '연결'을 담당하는 어댑터입니다. 우리는 `spring-cloud-stream-binder-kafka`를 사용할 것입니다.
2.  **함수 (Function):** 우리의 비즈니스 로직 그 자체입니다.
      * **`Consumer<T>`:** 이벤트를 '소비'만 하는 함수. `(T) -> Unit`. (예: `shipping-service`의 이벤트 처리 로직)
      * **`Supplier<T>`:** 이벤트를 '생산'만 하는 함수. `() -> T`. (예: 1초마다 센서 데이터를 보내는 로직)
      * **`Function<T, R>`:** 이벤트를 '가공'하여 새로운 이벤트를 만드는 함수. `(T) -> R`. (예: 원본 로그 이벤트를 받아 파싱된 로그 이벤트를 만드는 로직)
3.  **바인딩 (Binding):** `application.yml` 파일을 통해, 우리의 '함수'를 특정 '이벤트 채널(Kafka 토픽)'에 \*\*연결(Bind)\*\*하는 과정입니다.

-----

### Spring Cloud Stream 적용 맛보기

#### 1\. 의존성 추가 (`shipping-service` 기준)

```kotlin
// shipping-service/build.gradle.kts
dependencies {
    // 1. Spring Cloud Stream 스타터
    implementation("org.springframework.cloud:spring-cloud-stream")
    // 2. Kafka 바인더 추가
    implementation("org.springframework.cloud:spring-cloud-stream-binder-kafka")
    // ...
}
```

#### 2\. 이벤트 소비자(Consumer) 로직 작성 (`shipping-service`)

`KafkaConsumer` 같은 Kafka 전용 클래스 대신, 순수한 코틀린 함수를 `@Bean`으로 등록합니다.

```kotlin
// shipping-service/src/main/kotlin/com/ecommerce/shipping/service/ShippingService.kt
package com.ecommerce.shipping.service

@Configuration
class ShippingEventConsumer {
    
    private val log = LoggerFactory.getLogger(javaClass)

    /**
     * OrderPaidEvent를 소비하는 Consumer 함수를 Bean으로 등록
     * 이 함수는 Kafka에 대한 어떤 지식도 가지고 있지 않음!
     */
    @Bean
    fun onOrderPaid(): (OrderPaidEvent) -> Unit = { event ->
        log.info("OrderPaidEvent received: {}", event)
        // TODO: 이 이벤트를 기반으로 배송 시작 로직 구현
    }
}

// (OrderPaidEvent data class는 공통 모듈 또는 각 서비스에 정의)
data class OrderPaidEvent(val orderId: Long, val memberId: Long)
```

#### 3\. 바인딩 설정 (`shipping-service`의 `application.yml`)

`onOrderPaid`라는 이름의 함수를 `ecommerce.orders.paid`라는 Kafka 토픽에 '연결'합니다.

```yaml
# shipping-service/src/main/resources/application.yml
spring:
  cloud:
    stream:
      # Kafka 바인더를 위한 설정
      kafka:
        binder:
          brokers: localhost:9092 # Kafka 서버 주소
      
      # 함수와 토픽을 연결하는 바인딩 설정
      function:
        # 1. 활성화할 함수의 Bean 이름을 정의
        definition: onOrderPaid
      
      bindings:
        # 2. 'onOrderPaid' 함수의 '입력(in)' 채널에 대한 설정
        # 형식: <function-name>-<in/out>-<index>
        onOrderPaid-in-0:
          # 3. 이 채널이 구독할 Kafka 토픽의 이름
          destination: ecommerce.orders.paid
          # 4. Kafka 컨슈머 그룹 설정 (동일 그룹 내에서는 하나의 컨슈머만 메시지를 받음)
          group: shipping-service-group
```

이제 `shipping-service`를 실행하면, Spring Cloud Stream은 `onOrderPaid`라는 함수를 찾아, `ecommerce.orders.paid` 토픽을 구독하는 Kafka 컨슈머를 **자동으로 생성하고 실행**합니다.

우리의 비즈니스 로직은 특정 메시징 기술로부터 완전히 자유로워졌습니다. 이제 이 프레임워크를 사용하여, 다음 절에서는 실제 '이벤트 발행(Producer)' 로직을 `order-service`에 구현해 보겠습니다.