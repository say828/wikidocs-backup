## '주문 생성' 이벤트 발행(Publish): '주문' 서비스가 이벤트를 발행하는 과정

이제 `Spring Cloud Stream`의 추상화를 활용하여, `order-service`가 '주문 결제 완료'라는 비즈니스 이벤트를 Kafka로 발행(Publish)하는 실제 코드를 구현해 보겠습니다.

-----

### 이벤트 발행을 위한 도구: `StreamBridge`

07장 03절에서 우리는 이벤트를 '소비'하기 위해 `Consumer` 함수를 `@Bean`으로 등록했습니다. 그렇다면 이벤트를 '발행'할 때는 `Supplier` 함수를 써야 할까요?

`Supplier`는 "1초마다 무언가를 계속 발행"하는 것처럼 주기적인 데이터 생성에는 적합하지만, "REST API 호출이 성공했을 때 딱 한 번 발행"하는 것과 같은 **비정기적이고 명령형(Imperative)인 코드 흐름**에는 적합하지 않습니다.

이런 경우를 위해, Spring Cloud Stream은 \*\*`StreamBridge`\*\*라는 유틸리티 클래스를 제공합니다. `StreamBridge`는 `@Service`나 `@RestController` 같은 일반적인 스프링 컴포넌트에 주입하여, 원하는 시점에 원하는 이벤트를 원하는 채널로 손쉽게 보낼 수 있게 해줍니다.

-----

### `order-service`에 이벤트 발행 로직 구현하기

#### 1\. 의존성 확인

`order-service`의 `build.gradle.kts`에 `spring-cloud-stream`과 `spring-cloud-stream-binder-kafka` 의존성이 포함되어 있는지 확인합니다.

#### 2\. 이벤트 페이로드(Payload) 정의

발행할 이벤트의 데이터 구조를 `data class`로 정의합니다. 이 이벤트는 `shipping-service`나 다른 소비자들이 필요로 할 모든 정보를 담고 있어야 합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/event/OrderPaidEvent.kt
package com.ecommerce.order.event

import java.math.BigDecimal

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

#### 3\. 출력(Output) 바인딩 설정 (`application.yml`)

`order-service`가 이벤트를 발행할 '출력 채널'을 `application.yml`에 정의합니다.

```yaml
# order-service/src/main/resources/application.yml
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
      
      # 함수가 아닌 StreamBridge를 사용할 것이므로 function.definition은 필요 없음
      
      bindings:
        # 1. 이 '출력 바인딩'의 고유한 이름. StreamBridge에서 이 이름을 사용함.
        # 형식: <binding-name>-<in/out>-<index>
        orderPaidEventProducer-out-0:
          # 2. 이 바인딩이 연결될 Kafka 토픽 이름
          destination: ecommerce.orders.paid
```

  * **`<binding-name>`**: `orderPaidEventProducer`는 우리가 임의로 정한 논리적인 바인딩의 이름입니다. 코드의 가독성을 위해 이벤트 이름이나 목적을 담아 짓는 것이 좋습니다.

#### 4\. `OrderService`에서 `StreamBridge`를 사용하여 이벤트 발행

이제 `OrderService`의 '주문 생성 및 결제 처리' 로직 마지막 부분에, `StreamBridge`를 사용하여 이벤트를 발행하는 코드를 추가합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/service/OrderService.kt
package com.ecommerce.order.service

import com.ecommerce.order.event.OrderPaidEvent
import org.springframework.cloud.stream.function.StreamBridge // 1. StreamBridge import
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val productServiceClient: ProductServiceClient,
    private val streamBridge: StreamBridge, // 2. StreamBridge 주입
) {

    @Transactional
    fun createAndPayOrder(memberId: Long, request: CreateOrderRequest): OrderResponse {
        // ... (06장에서 구현한 상품 재고 확인 및 주문 생성 로직) ...
        val savedOrder = orderRepository.save(newOrder)

        // ... (결제 서비스 연동 로직) ...
        // paymentService.processPayment(...)
        // 결제가 성공했다고 가정
        savedOrder.markAsPaid()
        
        // --- 3. 이벤트 생성 ---
        val event = OrderPaidEvent(
            orderId = savedOrder.id!!,
            memberId = savedOrder.memberId,
            totalAmount = savedOrder.totalAmount.toBigDecimal(),
            orderLines = savedOrder.orderLines.map { 
                OrderPaidEvent.OrderLineItem(it.productId, it.quantity) 
            }
        )
        
        // --- 4. StreamBridge로 이벤트 발행 ---
        // 첫 번째 인자: application.yml에 정의한 바인딩 이름
        // 두 번째 인자: 보낼 이벤트 객체
        val isSent = streamBridge.send("orderPaidEventProducer-out-0", event)
        log.info("OrderPaidEvent sent for orderId {}: {}", savedOrder.id, isSent)

        return savedOrder.toResponse()
    }
}
```

-----

### 🤔 남겨진 과제: 트랜잭션과 메시지 발행의 원자성

위 코드는 잘 동작하지만, 사실 분산 시스템에서 매우 어려운 문제를 내포하고 있습니다.

> **"만약 `orderRepository.save()`의 DB 트랜잭션은 성공(Commit)했는데, `streamBridge.send()`가 Kafka 장애로 실패하면 어떻게 될까?"**

이 경우, 우리 시스템에는 '주문'은 존재하지만, 이 주문에 대한 '이벤트'는 발행되지 않아 `shipping-service`는 영원히 이 주문의 존재를 알 수 없게 됩니다. 데이터 정합성이 깨지는 것입니다.

DB 트랜잭션과 메시지 발행은 별개의 작업이므로 하나의 원자적인(Atomic) 트랜잭션으로 묶을 수 없습니다. 이 문제를 해결하기 위한 가장 견고한 패턴이 바로 **'트랜잭셔널 아웃박스(Transactional Outbox)'** 패턴이며, 이는 **11장: 서비스별 데이터베이스와 데이터 일관성**에서 심도 있게 다룰 것입니다.

우선 지금은, `order-service`가 성공적으로 이벤트를 발행하는 'Happy Path'를 완성했습니다. 다음 절에서는 이 이벤트를 `shipping-service`가 구독하여 처리하는 과정을 구현해 보겠습니다.