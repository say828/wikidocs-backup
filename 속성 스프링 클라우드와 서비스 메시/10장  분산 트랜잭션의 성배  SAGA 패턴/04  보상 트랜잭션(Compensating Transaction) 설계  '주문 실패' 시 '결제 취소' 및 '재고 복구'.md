## 보상 트랜잭션(Compensating Transaction) 설계: '주문 실패' 시 '결제 취소' 및 '재고 복구'

SAGA 패턴의 진정한 힘은 '성공' 경로가 아니라 '실패' 경로를 얼마나 견고하게 설계했느냐에 달려있습니다. SAGA의 안정성은 \*\*보상 트랜잭션(Compensating Transaction)\*\*의 설계에 의해 결정됩니다.

보상 트랜잭션은 단순한 DB `ROLLBACK`이 아닙니다. 이미 `COMMIT`되어버린 로컬 트랜잭션의 효과를, 비즈니스 로직을 통해 **의미적으로(Semantically)** 되돌리는 새로운 트랜잭션입니다.

이 절에서는 SAGA의 가장 중요한 부분인 보상 트랜잭션을 '주문 실패' 시나리오에 맞춰 설계해 보겠습니다.

-----

### 실패 시나리오 정의

우리의 '주문 생성' SAGA는 다음과 같은 단계를 거칩니다.

1.  `Order Service`: 주문 생성 (`PENDING`) → `OrderCreatedEvent` 발행
2.  `Payment Service`: 결제 처리 → `PaymentCompletedEvent` 발행
3.  `Product Service`: 재고 차감 → `StockDecreasedEvent` 발행

여기서 **3단계, '재고 차감'이 실패**했다고 가정해 봅시다. (예: 마지막 남은 재고를 다른 사용자가 동시에 구매하여 재고 부족 발생)

이때 `product-service`는 성공 이벤트 대신 \*\*`StockDecreaseFailedEvent`\*\*를 발행해야 합니다. 이 실패 이벤트가 바로 보상 트랜잭션들을 연쇄적으로 트리거하는 **신호탄**이 됩니다.

`StockDecreaseFailedEvent`는 보상 트랜잭션에 필요한 충분한 정보를 담고 있어야 합니다.

```kotlin
// 공통 이벤트 모듈 또는 각 서비스에 정의
data class StockDecreaseFailedEvent(
    val orderId: Long,
    val memberId: Long,
    val reason: String, // 실패 사유 (예: "OUT_OF_STOCK")
    val orderLines: List<OrderLineItem>, // 재고 복구를 위한 상품 정보
) {
    data class OrderLineItem(val productId: Long, val quantity: Int)
}
```

-----

### 보상 트랜잭션 구현 (코레오그래피 방식)

실패 이벤트가 발행되면, 관련 서비스들은 이 이벤트를 구독하여 각자의 보상 트랜잭션을 수행해야 합니다.

#### 1\. `Payment Service`: 결제 취소(환불)

`payment-service`는 `StockDecreaseFailedEvent`를 구독하고, 이미 성공시킨 '결제'를 '취소(환불)'하는 보상 트랜잭션을 실행해야 합니다.

```kotlin
// payment-service/src/main/kotlin/com/ecommerce/payment/consumer/PaymentEventConsumer.kt
@Component
class PaymentEventConsumer(
    private val paymentService: PaymentService
) {
    /**
     * 재고 차감 실패 이벤트를 구독하여 결제 환불 보상 트랜잭션을 실행
     */
    @KafkaListener(topics = ["ecommerce.stock.decrease.failed"], groupId = "payment-compensation-group")
    fun handleStockDecreaseFailed(event: StockDecreaseFailedEvent) {
        log.info("StockDecreaseFailedEvent received. Starting payment refund for orderId: {}", event.orderId)
        try {
            paymentService.refundPayment(event.orderId)
        } catch (e: Exception) {
            log.error("Failed to refund payment for orderId: ${event.orderId}", e)
            // DLQ 처리 등을 위해 예외를 다시 던짐
            throw e
        }
    }
}

// payment-service/src/main/kotlin/com/ecommerce/payment/service/PaymentService.kt
@Service
class PaymentService(...) {
    @Transactional
    fun refundPayment(orderId: Long) {
        // 1. 해당 주문의 결제 기록 조회
        val payment = paymentRepository.findByOrderId(orderId) ?:
            throw EntityNotFoundException("결제 정보를 찾을 수 없습니다.")

        // 2. 멱등성(Idempotency) 체크: 이미 환불되었는지 확인
        if (payment.status == PaymentStatus.REFUNDED) {
            log.warn("Payment for orderId {} has already been refunded.", orderId)
            return
        }

        // 3. 외부 PG(Payment Gateway) 환불 API 호출
        // pgClient.refund(payment.pgTransactionId)

        // 4. 서비스 내부 DB 상태를 REFUNDED로 변경
        payment.markAsRefunded()
        paymentRepository.save(payment)
        log.info("Payment for orderId {} has been successfully refunded.", orderId)
    }
}
```

#### 2\. `Order Service`: 주문 취소

`order-service` 역시 `StockDecreaseFailedEvent`를 구독하여, 최초에 생성했던 `Order`의 상태를 `CANCELLED`로 변경하는 보상 트랜잭션을 실행해야 합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/consumer/OrderEventConsumer.kt
@Component
class OrderEventConsumer(
    private val orderService: OrderService
) {
    /**
     * 재고 차감 실패 이벤트를 구독하여 주문 취소 보상 트랜잭션을 실행
     */
    @KafkaListener(topics = ["ecommerce.stock.decrease.failed"], groupId = "order-compensation-group")
    fun handleStockDecreaseFailed(event: StockDecreaseFailedEvent) {
        log.info("StockDecreaseFailedEvent received. Cancelling order for orderId: {}", event.orderId)
        orderService.cancelOrderForSaga(event.orderId, event.reason)
    }
}

// order-service/src/main/kotlin/com/ecommerce/order/service/OrderService.kt
@Service
class OrderService(...) {
    @Transactional
    fun cancelOrderForSaga(orderId: Long, reason: String) {
        val order = orderRepository.findByIdOrNull(orderId) ?:
            throw EntityNotFoundException("주문을 찾을 수 없습니다.")
        
        // 멱등성 체크 및 비즈니스 로직은 Order 애그리거트가 담당
        order.cancel(reason) // 예: cancel(reason: String) 메서드
        orderRepository.save(order)
        log.info("Order for orderId {} has been successfully cancelled due to: {}", orderId, reason)
    }
}
```

-----

### 보상 트랜잭션 설계의 핵심 원칙

  * **멱등성(Idempotency):** 보상 트랜잭션은 **반드시 멱등성을 보장**해야 합니다. Kafka와 같은 메시지 시스템은 동일한 이벤트를 중복으로 전달할 수 있습니다. `refundPayment` 로직이 여러 번 호출되더라도, 실제 환불은 단 한 번만 일어나도록 상태를 확인하는 로직이 필수입니다.
  * **신뢰성:** 보상 트랜잭션 자체는 실패할 확률이 거의 없어야 합니다. (예: '환불' 로직이 복잡한 비즈니스 검증 때문에 실패해서는 안 됨)
  * **교환성(Commutativity):** 가능하다면 보상 트랜잭션의 실행 순서가 결과에 영향을 주지 않도록 설계하는 것이 좋습니다. 우리 예제에서는 '결제 취소'와 '주문 취소'가 동시에 일어나도 문제가 없습니다.

견고한 보상 트랜잭션을 설계하는 것은 SAGA 패턴 구현에서 가장 어렵고 중요한 부분입니다. 모든 실패 경로를 신중하게 고려하고, 각 단계에 대한 명확한 '되돌리기' 정책을 수립함으로써, 우리는 장애 상황에서도 스스로를 치유하고 데이터 일관성을 유지하는 탄력적인 시스템을 만들 수 있습니다.