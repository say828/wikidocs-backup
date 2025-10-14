## Spring Kafka를 활용한 이커머스 이벤트 파이프라인 구축 (실전 코드)

지금까지 Kafka의 이론과 내부 아키텍처를 깊이 있게 학습했습니다. 이제 이 지식을 바탕으로, `Spring Cloud Stream`이라는 추상화 계층 뒤에서 벗어나 **`Spring Kafka`** 라이브러리를 직접 사용하여 우리의 이커머스 이벤트 파이프라인을 더욱 정교하게 제어해 보겠습니다.

`Spring Cloud Stream`이 브로커에 구애받지 않는 유연성을 제공한다면, `Spring Kafka`는 Kafka의 특정 기능(수동 오프셋 커밋, 파티션 제어, 레코드 헤더 등)을 직접 활용해야 할 때 강력한 힘을 발휘합니다.

-----

### 1\. 이벤트 발행(Producer) 구현 (`order-service`)

#### 1-1. 의존성 및 `application.yml` 설정

`order-service`의 `build.gradle.kts`에 `spring-kafka` 의존성을 추가합니다.

```kotlin
// order-service/build.gradle.kts
dependencies {
    implementation("org.springframework.kafka:spring-kafka")
    // ...
}
```

`application.yml`에 프로듀서 관련 설정을 추가합니다. 특히, 데이터 유실을 막기 위한 `acks: all` 설정을 명시합니다.

```yaml
# order-service/src/main/resources/application.yml
spring:
  kafka:
    producer:
      bootstrap-servers: localhost:9092
      # 1. 메시지 키는 String 타입 (orderId)
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      # 2. 메시지 값은 JSON 형태로 직렬화
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      properties:
        # 3. 데이터 무결성을 위한 acks=all 설정
        acks: all 
```

#### 1-2. `KafkaTemplate`을 이용한 프로듀서 코드 작성

`Spring Kafka`는 `KafkaTemplate`이라는 편리한 클래스를 제공하여 메시지 전송을 단순화합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/service/OrderEventProducer.kt
package com.ecommerce.order.service

import com.ecommerce.order.event.OrderPaidEvent
import org.slf4j.LoggerFactory
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.stereotype.Service

@Service
class OrderEventProducer(
    // 1. KafkaTemplate 주입 (Key: String, Value: OrderPaidEvent)
    private val kafkaTemplate: KafkaTemplate<String, OrderPaidEvent>
) {
    private val log = LoggerFactory.getLogger(javaClass)
    private val topic = "ecommerce.orders.paid"

    fun sendOrderPaidEvent(event: OrderPaidEvent) {
        try {
            // 2. 토픽, 파티셔닝 키(orderId), 이벤트 페이로드를 담아 메시지 전송
            // orderId를 키로 사용하여 동일 주문에 대한 이벤트는 항상 같은 파티션으로 가도록 보장
            val future = kafkaTemplate.send(topic, event.orderId.toString(), event)
            
            future.whenComplete { result, ex ->
                if (ex != null) {
                    log.error("Failed to send message for orderId ${event.orderId}", ex)
                } else {
                    log.info("Successfully sent message for orderId {}: offset={}", 
                        event.orderId, result.recordMetadata.offset())
                }
            }
        } catch (e: Exception) {
            log.error("Exception while sending message for orderId ${event.orderId}", e)
        }
    }
}
```

  * `OrderService`는 이제 `StreamBridge` 대신 이 `OrderEventProducer`를 주입받아 이벤트를 발행하면 됩니다.

-----

### 2\. 이벤트 구독(Consumer) 구현 (`shipping-service`)

#### 2-1. 의존성 및 `application.yml` 설정

`shipping-service`에도 `spring-kafka` 의존성을 추가하고, `application.yml`에 컨슈머 설정을 추가합니다.

```yaml
# shipping-service/src/main/resources/application.yml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:9092
      # 1. 컨슈머 그룹 ID
      group-id: shipping-service-group
      # 2. 새로운 컨슈머 그룹일 경우, 토픽의 가장 처음부터 메시지를 읽어옴
      auto-offset-reset: earliest
      # 3. 메시지 키 역직렬화
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      # 4. 메시지 값(JSON) 역직렬화
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        # 5. JsonDeserializer가 역직렬화할 객체의 패키지를 신뢰하도록 설정
        spring.json.trusted.packages: "com.ecommerce.shipping.event"
```

#### 2-2. `@KafkaListener`를 이용한 컨슈머 코드 작성

`Spring Kafka`의 `@KafkaListener` 어노테이션을 사용하면, 복잡한 컨슈머 루프(loop) 로직 없이 간단하게 메시지를 수신할 수 있습니다.

```kotlin
// shipping-service/src/main/kotlin/com/ecommerce/shipping/consumer/ShippingEventConsumer.kt
package com.ecommerce.shipping.consumer

import com.ecommerce.shipping.event.OrderPaidEvent
import com.ecommerce.shipping.service.ShippingService
import org.slf4j.LoggerFactory
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.kafka.support.KafkaHeaders
import org.springframework.messaging.handler.annotation.Header
import org.springframework.messaging.handler.annotation.Payload
import org.springframework.stereotype.Component

@Component
class ShippingEventConsumer(
    private val shippingService: ShippingService
) {
    private val log = LoggerFactory.getLogger(javaClass)

    /**
     * @KafkaListener 어노테이션을 통해 특정 토픽과 그룹의 메시지를 구독
     */
    @KafkaListener(
        topics = ["ecommerce.orders.paid"], 
        groupId = "shipping-service-group"
    )
    fun handleOrderPaidEvent(
        @Payload event: OrderPaidEvent, // 1. 메시지 페이로드를 OrderPaidEvent 객체로 자동 역직렬화
        @Header(KafkaHeaders.RECEIVED_KEY) key: String, // 2. 메시지 키(orderId)
        @Header(KafkaHeaders.RECEIVED_PARTITION) partition: Int,
        @Header(KafkaHeaders.OFFSET) offset: Long
    ) {
        log.info("Received event: {}, key={}, partition={}, offset={}", 
            event, key, partition, offset)
        
        try {
            // 3. 비즈니스 로직 처리
            shippingService.startShipping(event)
        } catch (e: Exception) {
            log.error("Error processing event for orderId ${event.orderId}", e)
            // 에러 발생 시, Spring Kafka의 기본 에러 핸들러가 동작하여 재시도 또는 DLQ로 보냄
            throw e 
        }
    }
}
```

-----

이것으로 08장을 마칩니다. 우리는 Kafka의 핵심 아키텍처(토픽, 파티션, 오프셋)와 프로듀서/컨슈머의 작동 원리(acks, 리밸런싱)를 깊이 있게 이해했으며, `Spring Kafka`를 통해 실제 코드 레벨에서 강력하고 안정적인 이벤트 파이프라인을 구축했습니다.

이제 우리는 대규모 트래픽을 처리할 수 있는 견고한 비동기 통신 기반을 갖추었습니다. 다음 09장에서는 이 기반 위에서, 이커머스 시스템의 고질적인 문제인 '읽기/쓰기 성능'을 최적화하기 위한 **CQRS 패턴**을 도입해 보겠습니다.