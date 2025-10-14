## Spring을 이용한 코레오그래피 기반 '주문' SAGA 구현 (실전 코드)

지금까지 SAGA 패턴의 모든 이론(코레오그래피 vs 오케스트레이션, 보상 트랜잭션)을 학습했습니다. 이제 이 모든 조각을 하나로 모아, **코레오그래피(Choreography)** 방식을 기반으로 우리 이커머스 프로젝트의 '주문 생성' 분산 트랜잭션을 실제 코드로 완성해 보겠습니다.

우리는 08장에서 구현한 `Spring Kafka`를 이벤트 채널로 사용하여 서비스 간의 협업을 구현합니다.

-----

### SAGA 흐름 요약

  * **주요 토픽(Topics):**
      * `ecommerce.orders.created`
      * `ecommerce.payments.completed`
      * `ecommerce.stock.decreased`
      * `ecommerce.stock.decrease.failed` (보상 트랜잭션 트리거)
  * **성공 경로 (Happy Path):**
    1.  `order-service`: `OrderCreatedEvent` 발행
    2.  `payment-service`: `OrderCreatedEvent` 수신 → `PaymentCompletedEvent` 발행
    3.  `product-service`: `PaymentCompletedEvent` 수신 → `StockDecreasedEvent` 발행
    4.  `order-service`: `StockDecreasedEvent` 수신 → 주문 상태 `CONFIRMED`로 변경
  * **실패 경로 (Failure Path):**
      * ... 3단계에서 `product-service`가 `StockDecreaseFailedEvent` 발행
      * `payment-service`: `StockDecreaseFailedEvent` 수신 → **결제 환불** (보상)
      * `order-service`: `StockDecreaseFailedEvent` 수신 → **주문 취소** (보상)

-----

### 1단계: `order-service` - SAGA 시작점

`OrderService`는 SAGA를 시작하는 `OrderCreatedEvent`를 발행합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/service/OrderService.kt
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val orderEventProducer: OrderEventProducer, // Kafka 프로듀서
) {
    @Transactional
    fun createOrder(command: CreateOrderCommand): Long {
        // 1. 주문을 'PENDING' 상태로 생성하고 DB에 저장
        val order = Order(
            memberId = command.memberId,
            productId = command.productId,
            status = OrderStatus.PENDING,
            // ...
        )
        val savedOrder = orderRepository.save(order)
        val orderId = savedOrder.id!!

        // 2. SAGA를 시작하는 첫 이벤트를 발행
        val event = OrderCreatedEvent(orderId = orderId, /* ... */)
        orderEventProducer.sendOrderCreatedEvent(event)
        log.info("OrderCreatedEvent sent for orderId: {}", orderId)
        
        return orderId
    }
}
```

-----

### 2단계: `payment-service` - 결제 처리 및 이벤트 발행

`payment-service`는 `OrderCreatedEvent`를 구독하여 결제를 처리하고, 다음 단계를 위한 `PaymentCompletedEvent`를 발행합니다.

```kotlin
// payment-service/src/main/kotlin/com/ecommerce/payment/consumer/PaymentEventConsumer.kt
@Component
class PaymentEventConsumer(
    private val paymentService: PaymentService
) {
    @KafkaListener(topics = ["ecommerce.orders.created"], groupId = "payment-group")
    fun handleOrderCreated(event: OrderCreatedEvent) {
        log.info("OrderCreatedEvent received. Starting payment for orderId: {}", event.orderId)
        paymentService.processPayment(event)
    }
}

// payment-service/src/main/kotlin/com/ecommerce/payment/service/PaymentService.kt
@Service
class PaymentService(
    private val paymentRepository: PaymentRepository,
    private val paymentEventProducer: PaymentEventProducer, // Kafka 프로듀서
) {
    @Transactional
    fun processPayment(event: OrderCreatedEvent) {
        // ... 외부 PG 연동 결제 처리 로직 ...
        // 결제가 성공했다고 가정
        
        val payment = Payment(orderId = event.orderId, status = PaymentStatus.COMPLETED)
        paymentRepository.save(payment)

        // SAGA의 다음 단계를 위한 이벤트 발행
        val nextEvent = PaymentCompletedEvent(orderId = event.orderId, /* ... */)
        paymentEventProducer.sendPaymentCompletedEvent(nextEvent)
        log.info("PaymentCompletedEvent sent for orderId: {}", event.orderId)
    }
}
```

-----

### 3단계: `product-service` - 재고 차감 및 분기 처리

`product-service`는 SAGA의 분기점입니다. 재고 차감 성공 여부에 따라 다른 이벤트를 발행합니다.

```kotlin
// product-service/src/main/kotlin/com/ecommerce/product/consumer/ProductEventConsumer.kt
@Component
class ProductEventConsumer(
    private val productService: ProductService
) {
    @KafkaListener(topics = ["ecommerce.payments.completed"], groupId = "product-group")
    fun handlePaymentCompleted(event: PaymentCompletedEvent) {
        log.info("PaymentCompletedEvent received. Decreasing stock for orderId: {}", event.orderId)
        productService.decreaseStockForSaga(event)
    }
}

// product-service/src/main/kotlin/com/ecommerce/product/service/ProductService.kt
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val productEventProducer: ProductEventProducer, // Kafka 프로듀서
) {
    @Transactional
    fun decreaseStockForSaga(event: PaymentCompletedEvent) {
        try {
            val product = productRepository.findByIdOrNull(event.productId) ?: throw Exception("상품 없음")
            
            // 재고 차감 로직 (실패 가능성 있음)
            product.decreaseStock(event.quantity) 
            productRepository.save(product)

            // 성공 시: StockDecreasedEvent 발행
            val successEvent = StockDecreasedEvent(orderId = event.orderId, /* ... */)
            productEventProducer.sendStockDecreasedEvent(successEvent)
            log.info("StockDecreasedEvent sent for orderId: {}", event.orderId)

        } catch (e: Exception) {
            // 실패 시: StockDecreaseFailedEvent 발행 (보상 트랜잭션 트리거)
            log.error("Failed to decrease stock for orderId: ${event.orderId}", e)
            val failureEvent = StockDecreaseFailedEvent(orderId = event.orderId, reason = e.message, /* ... */)
            productEventProducer.sendStockDecreaseFailedEvent(failureEvent)
        }
    }
}
```

-----

### 4단계: `order-service` - SAGA 완료 처리

`order-service`는 성공 이벤트(`StockDecreasedEvent`)를 구독하여 주문 상태를 최종 업데이트합니다. (실패 이벤트 구독 로직은 10장 04절에서 이미 구현)

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/consumer/OrderEventConsumer.kt
@Component
class OrderEventConsumer(
    private val orderService: OrderService
) {
    // ... (handleStockDecreaseFailed 메서드) ...

    @KafkaListener(topics = ["ecommerce.stock.decreased"], groupId = "order-group")
    fun handleStockDecreased(event: StockDecreasedEvent) {
        log.info("StockDecreasedEvent received. Confirming order for orderId: {}", event.orderId)
        orderService.confirmOrder(event.orderId)
    }
}

// order-service/src/main/kotlin/com/ecommerce/order/service/OrderService.kt
@Service
class OrderService(...) {
    // ... (createOrder, cancelOrderForSaga 메서드) ...
    
    @Transactional
    fun confirmOrder(orderId: Long) {
        val order = orderRepository.findByIdOrNull(orderId) ?: return
        order.confirm() // 주문 상태를 CONFIRMED로 변경
        orderRepository.save(order)
        log.info("Order for orderId {} has been confirmed.", orderId)
    }
}
```

이것으로 SAGA의 전체 흐름이 완성되었습니다. 각 서비스는 Kafka 이벤트를 통해 서로 느슨하게 연결되어, 중앙 지휘자 없이도 성공 경로와 실패/보상 경로를 따라 자율적으로 협업하여 분산 트랜잭션을 완료합니다.

10장을 마칩니다. 우리는 분산 시스템의 가장 어려운 숙제 중 하나인 '분산 트랜잭션'을 SAGA 패턴과 Kafka를 이용해 해결했습니다. 다음 11장에서는 각 서비스가 자신만의 DB를 가질 때 발생하는 '데이터 일관성' 문제를 더 깊이 파고들어 보겠습니다.