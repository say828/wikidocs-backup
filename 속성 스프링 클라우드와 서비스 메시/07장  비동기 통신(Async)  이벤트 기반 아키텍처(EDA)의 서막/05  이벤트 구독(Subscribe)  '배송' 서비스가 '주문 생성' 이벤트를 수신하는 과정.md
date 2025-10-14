## 이벤트 구독(Subscribe): '배송' 서비스가 '주문 생성' 이벤트를 수신하는 과정

07장 04절에서 `order-service`는 성공적으로 `OrderPaidEvent`를 Kafka 토픽으로 발행했습니다. 이제 이 여정의 마지막 조각을 맞출 차례입니다. `shipping-service`가 이 이벤트를 구독(Subscribe)하여, '배송 시작'이라는 실제 비즈니스 로직을 트리거하는 과정을 구현합니다.

07장 03절에서 이미 `Spring Cloud Stream`을 이용한 소비자(Consumer)의 기본 구조를 살펴보았으므로, 여기서는 실제 `shipping-service`의 코드를 완성하는 데 집중합니다.

-----

### `shipping-service`에 이벤트 구독 로직 구현하기

#### 1\. 의존성 및 이벤트 DTO 확인

`shipping-service`의 `build.gradle.kts`에 `spring-cloud-stream-binder-kafka` 의존성이 있는지 확인합니다.

가장 중요한 것은, 생산자인 `order-service`가 발행하는 `OrderPaidEvent`와 **정확히 동일한 구조**의 `data class`를 `shipping-service`도 가지고 있어야 한다는 점입니다.

```kotlin
// shipping-service/src/main/kotlin/com/ecommerce/shipping/event/OrderPaidEvent.kt
package com.ecommerce.shipping.event

import java.math.BigDecimal

// order-service의 이벤트 DTO와 동일한 'API 계약'
data class OrderPaidEvent(
    val orderId: Long,
    val memberId: Long,
    val totalAmount: BigDecimal,
    val orderLines: List<OrderLineItem>,
) {
    data class OrderLineItem(
        val productId: Long,
        val quantity: Int,
    )
}
```

> ***저자 주:*** *실제 프로덕션 프로젝트에서는 이처럼 중요한 DTO를 각 서비스에 복사-붙여넣기 하는 대신, `common-events.jar`와 같은 **공유 라이브러리**로 만들어 양쪽 서비스가 모두 의존하도록 관리하는 것이 بهترین practice 입니다. 이는 서비스 간의 'API 계약'이 깨지는 것을 방지합니다.*

#### 2\. `application.yml` 바인딩 설정

`shipping-service`가 어떤 토픽을, 어떤 그룹으로 구독할지 `application.yml`에 명시합니다. 이는 07장 03절에서 살펴본 내용과 동일합니다.

```yaml
# shipping-service/src/main/resources/application.yml
spring:
  application:
    name: shipping-service
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
      function:
        definition: onOrderPaid # 1. 활성화할 Consumer 함수 Bean 이름
      bindings:
        onOrderPaid-in-0:
          destination: ecommerce.orders.paid # 2. 구독할 Kafka 토픽
          group: shipping-service-group      # 3. 컨슈머 그룹 ID
```

  * **컨슈머 그룹 (`group`):** 이는 Kafka의 매우 중요한 개념입니다. `shipping-service`가 여러 대(인스턴스)로 확장되더라도, **동일한 `group` ID를 사용하는 컨슈머들에게는 하나의 이벤트가 단 한 번만 전달됨**을 보장합니다. 즉, '주문 123'에 대한 배송이 두 번 시작되는 일이 없습니다.

#### 3\. Consumer 함수 및 비즈니스 로직 구현

이제 이벤트를 받아 실제 '배송' 로직을 처리하는 코드를 작성합니다.

```kotlin
// shipping-service/src/main/kotlin/com/ecommerce/shipping/service/ShippingService.kt
package com.ecommerce.shipping.service

import com.ecommerce.shipping.domain.Shipment
import com.ecommerce.shipping.domain.ShipmentRepository
import com.ecommerce.shipping.event.OrderPaidEvent
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class ShippingService(
    private val shipmentRepository: ShipmentRepository
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @Transactional
    fun startShipping(event: OrderPaidEvent) {
        log.info("Starting shipping process for orderId: {}", event.orderId)
        
        // 1. 이미 처리된 이벤트인지 멱등성(Idempotency) 체크 (선택적이지만 중요)
        if (shipmentRepository.existsByOrderId(event.orderId)) {
            log.warn("Shipping process for orderId {} already started.", event.orderId)
            return
        }
        
        // 2. 이벤트 정보를 바탕으로 Shipment 엔티티 생성
        val shipment = Shipment(
            orderId = event.orderId,
            memberId = event.memberId,
            // ... 배송지 정보 등은 이벤트를 더 풍부하게 만들거나,
            // order-service에 API를 동기 호출하여 조회할 수도 있음 ...
        )
        
        // 3. DB에 저장
        shipmentRepository.save(shipment)
        log.info("Shipment created for orderId: {}", event.orderId)
    }
}
```

```kotlin
// shipping-service/src/main/kotlin/com/ecommerce/shipping/consumer/ShippingEventConsumer.kt
package com.ecommerce.shipping.consumer

import com.ecommerce.shipping.event.OrderPaidEvent
import com.ecommerce.shipping.service.ShippingService
import org.slf4j.LoggerFactory
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class ShippingEventConsumer(
    private val shippingService: ShippingService // 1. 비즈니스 로직을 처리할 서비스 주입
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @Bean
    fun onOrderPaid(): (OrderPaidEvent) -> Unit = { event ->
        log.info(">> Received OrderPaidEvent: {}", event)
        try {
            // 2. 서비스 레이어에 작업 위임
            shippingService.startShipping(event)
        } catch (e: Exception) {
            // 3. 에러 처리
            log.error("Failed to process shipping for orderId: ${event.orderId}", e)
            // 예외를 던지면 Spring Cloud Stream의 재시도/DLQ 메커니즘이 동작
            throw e 
        }
    }
}
```

-----

### 실패 처리: 재시도와 Dead Letter Queue (DLQ)

만약 `shippingService.startShipping(event)` 로직 수행 중 `shipping-service`의 DB 장애로 예외가 발생하면 어떻게 될까요?

Spring Cloud Stream Kafka Binder는 기본적으로 매우 견고한 실패 처리 메커니즘을 제공합니다.

1.  **재시도 (Retry):** 메시지 처리에 실패하면, 기본적으로 **3번** 더 재시도합니다. 일시적인 네트워크 문제라면 재시도 과정에서 성공할 수 있습니다.
2.  **데드 레터 큐 (Dead Letter Queue, DLQ):** 재시도 횟수를 모두 소진해도 계속 실패하는 '독이 든 메시지(Poison Pill)'는, **별도의 DLQ 토픽**으로 격리됩니다. (예: `error.ecommerce.orders.paid.shipping-service-group`)
      * 이는 '문제 있는 메시지' 하나 때문에 전체 파티션의 다른 정상적인 메시지 처리가 막히는 것을 방지하는 매우 중요한 기능입니다. 개발자는 나중에 DLQ에 쌓인 메시지를 분석하여 원인을 파악하고 수동으로 복구할 수 있습니다.

-----

이것으로 07장을 마칩니다. 우리는 `order-service`가 발행한 `OrderPaidEvent`를 `shipping-service`가 비동기적으로 구독하여 처리하는 완전한 EDA 흐름을 구축했습니다. 두 서비스는 메시지 브로커를 통해 완벽하게 분리되었으며, 한쪽의 장애가 다른 쪽에 직접적인 영향을 주지 않는 탄력적인 아키텍처를 완성했습니다.

지금까지 우리는 EDA와 Kafka를 '사용하는 법'에 초점을 맞추었습니다. 다음 08장에서는 한 걸음 더 깊이 들어가, Kafka가 어떻게 이런 놀라운 성능과 안정성을 제공하는지 그 **내부 아키텍처를 심층 분석**해 보겠습니다.