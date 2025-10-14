## 데이터 동기화 문제: Command 측의 변경을 어떻게 Query 측에 반영할 것인가?

우리는 이제 상태를 변경하는 효율적인 '명령(Command) 파이프라인'과, 데이터를 조회하는 초고속 '조회(Query) 파이프라인'을 모두 갖추었습니다. 하지만 이 둘은 현재 서로를 알지 못하는 별개의 세상에 존재합니다.

`OrderService`가 `CreateOrderCommand`를 처리하여 `orders` 테이블(쓰기 모델)에 새로운 주문을 성공적으로 저장했습니다. 하지만 `order_summaries` 테이블(읽기 모델)은 여전히 비어있습니다. 이 상태에서 사용자가 주문 내역을 조회하면 아무것도 보이지 않을 것입니다.

어떻게 **쓰기 모델의 변경 사항**을 **읽기 모델에 반영**할 수 있을까요? 이 데이터 동기화 문제를 해결하는 것이 CQRS 패턴의 마지막 화룡점정입니다.

-----

### 잘못된 접근법: 듀얼 라이트 (Dual Writes) ❌

가장 먼저 떠오르는 순진한 방법은, `OrderService`(CommandHandler)가 **하나의 트랜잭션 안에서 쓰기 모델과 읽기 모델 둘 다에 쓰는 것**입니다.

```kotlin
// (나쁜 예시) Command Handler가 Read Model을 직접 수정하는 경우
@Service
@Transactional
class OrderService(
    private val orderRepository: OrderRepository, // Write Repo
    private val orderSummaryRepository: OrderSummaryRepository // Read Repo
) {
    fun createOrder(command: CreateOrderCommand) {
        val order = Order(...)
        orderRepository.save(order) // 1. 쓰기 모델에 저장
        
        val summary = OrderSummary(...)
        orderSummaryRepository.save(summary) // 2. 읽기 모델에도 저장
    }
}
```

이 방식은 CQRS의 **모든 장점을 파괴하는 최악의 안티패턴**입니다.

  * **강한 결합:** 쓰기 모델이 읽기 모델의 스키마와 존재를 알아야 합니다. 만약 UI 변경으로 `OrderSummary`에 필드 하나가 추가되면, 핵심 비즈니스 로직이 담긴 `OrderService`를 수정해야 합니다.
  * **단일 책임 원칙 위배:** `OrderService`는 이제 '주문 생성'이라는 핵심 책임과 '주문 내역 조회 모델 갱신'이라는 부가적인 책임을 함께 갖게 되어 코드가 복잡해집니다.
  * **트랜잭션 부하:** 쓰기 트랜잭션이 읽기 모델 테이블까지 Lock을 걸게 되어 성능이 저하됩니다.

-----

### 올바른 해답: 이벤트 기반 동기화 (Event-Driven Synchronization) ✅

이 문제를 해결하는 가장 우아하고 확장성 있는 방법은 바로 07, 08장에서 구축한 \*\*이벤트 기반 아키텍처(EDA)\*\*를 활용하는 것입니다. 쓰기 모델과 읽기 모델은 **이벤트**를 통해 비동기적으로 대화합니다.

**전체 흐름:**

1.  **[Command]** `OrderService`는 Command를 처리하고, `orders` 테이블(쓰기 DB)에 **상태 변경을 커밋**합니다.
2.  **[Event]** DB 트랜잭션이 성공적으로 커밋되면, `OrderService`는 **`OrderCreatedEvent`** 같은 '상태 변경' 이벤트를 발행하여 **Kafka**로 보냅니다.
3.  **[Projection]** \*\*'프로젝터(Projector)'\*\*라고 불리는 이벤트 컨슈머가 이 이벤트를 구독합니다.
4.  **[Query Model Update]** 프로젝터는 이벤트에 담긴 데이터를 사용하여 `order_summaries` 테이블(읽기 DB)의 **읽기 모델을 생성하거나 갱신**합니다.

#### 프로젝터 (Projector) 구현

'프로젝터'는 이벤트를 받아 읽기 모델을 만드는 책임만 지는 컴포넌트입니다. `order-service` 내부에 Kafka 컨슈머로 구현할 수 있습니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/projection/OrderProjector.kt
package com.ecommerce.order.projection

import com.ecommerce.order.client.ProductServiceClient
import com.ecommerce.order.event.OrderCancelledEvent
import com.ecommerce.order.event.OrderCreatedEvent
import com.ecommerce.order.query.OrderSummary
import com.ecommerce.order.query.OrderSummaryRepository
import org.slf4j.LoggerFactory
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.stereotype.Component

@Component
class OrderProjector(
    private val orderSummaryRepository: OrderSummaryRepository,
    private val productServiceClient: ProductServiceClient // 1. 상품 정보 조회를 위한 Feign 클라이언트
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @KafkaListener(topics = ["ecommerce.orders.created"], groupId = "order-projection-group")
    fun onOrderCreated(event: OrderCreatedEvent) {
        log.info("Projecting OrderCreatedEvent: {}", event)
        
        try {
            // 2. 읽기 모델에 필요한 추가 정보(상품 이름 등)를 API로 조회
            val product = productServiceClient.getProduct(event.productId)
            
            // 3. 읽기 모델(OrderSummary) 생성
            val summary = OrderSummary(
                orderId = event.orderId,
                orderDate = event.orderDate,
                status = event.status,
                productId = event.productId,
                productName = product.name, // 조회해온 정보 사용
                productThumbnailUrl = product.thumbnailUrl, // 조회해온 정보 사용
                quantity = event.quantity,
                unitPrice = event.unitPrice,
                totalPrice = event.unitPrice.multiply(event.quantity.toBigDecimal()),
                memberId = event.memberId
            )
            
            // 4. 읽기 DB에 저장
            orderSummaryRepository.save(summary)
        } catch (e: Exception) {
            log.error("Error projecting OrderCreatedEvent for orderId ${event.orderId}", e)
            // (에러 처리 및 재시도 로직)
        }
    }

    @KafkaListener(topics = ["ecommerce.orders.cancelled"], groupId = "order-projection-group")
    fun onOrderCancelled(event: OrderCancelledEvent) {
        log.info("Projecting OrderCancelledEvent: {}", event)
        
        // 5. 기존 읽기 모델을 찾아 상태만 변경
        orderSummaryRepository.findById(event.orderId).ifPresent { summary ->
            summary.status = OrderStatus.CANCELLED
            orderSummaryRepository.save(summary)
        }
    }
}
```

  * **참고:** `OrderService`가 `OrderCreatedEvent`를 발행하는 로직은 11장에서 다룰 '트랜잭셔널 아웃박스' 패턴을 통해 DB 트랜잭션과 원자적으로 묶어 데이터 정합성을 보장할 것입니다.

-----

이것으로 09장을 마칩니다. 우리는 **CQRS와 EDA를 결합**하여 다음과 같은 이상적인 아키텍처를 완성했습니다.

  * **Command 측:** DDD 애그리거트를 통해 데이터의 일관성을 강력하게 보장하며 상태를 변경합니다.
  * **Query 측:** 비정규화된 읽기 전용 모델을 통해 사용자에게 초고속으로 데이터를 제공합니다.
  * **Event-Driven Sync:** 두 모델은 Kafka 이벤트를 통해 비동기적으로 동기화되어, 서로에게 영향을 주지 않고 독립적으로 확장하고 최적화될 수 있습니다.

이제 우리의 시스템은 복잡한 비즈니스 요구사항과 대규모 트래픽을 모두 감당할 수 있는 강력한 구조를 갖추게 되었습니다. 다음 10장에서는 분산 시스템의 가장 어려운 문제, **분산 트랜잭션**을 해결하기 위한 **SAGA 패턴**의 세계로 떠나보겠습니다.