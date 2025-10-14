## Command 모델 설계: '주문 생성/수정/취소' API 구현

이제 CQRS 패턴의 첫 번째 부분인 **명령(Command) 측 파이프라인**을 실제 코드로 구현해 보겠습니다. 이 파이프라인의 유일한 목적은 '주문' 애그리거트(Aggregate)의 상태를 '생성', '수정', '취소'하는 것입니다.

우리는 02장에서 설계한 `Order` 애그리거트를 \*\*쓰기 모델(Write Model)\*\*로 사용하여, 비즈니스 규칙의 일관성을 강력하게 보장할 것입니다.

-----

### 1\. Command 객체 정의

먼저, 상태 변경의 '의도'를 나타내는 Command 객체들을 `data class`로 정의합니다. 이 객체들은 비즈니스 로직 없이 데이터만 담고 있는 단순한 DTO입니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/command/OrderCommands.kt
package com.ecommerce.order.command

// 주문 생성을 위한 Command
data class CreateOrderCommand(
    val memberId: Long,
    val productId: Long,
    val quantity: Int,
    val shippingAddress: String,
)

// 배송지 변경을 위한 Command
data class UpdateShippingInfoCommand(
    val orderId: Long,
    val newShippingAddress: String,
)

// 주문 취소를 위한 Command
data class CancelOrderCommand(
    val orderId: Long,
)
```

-----

### 2\. 쓰기 모델(Write Model): `Order` 애그리거트 강화

우리의 `Order` 애그리거트가 Command를 처리할 비즈니스 로직(메서드)을 갖도록 강화합니다. 애그리거트 루트는 외부의 데이터를 받아 스스로의 상태를 변경하고, 비즈니스 규칙(불변성)을 검증하는 책임을 가집니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/domain/Order.kt
package com.ecommerce.order.domain

@Entity
@Table(name = "orders")
class Order(
    val memberId: Long,
    // ...
    var shippingAddress: String,
    var status: OrderStatus = OrderStatus.PENDING,
) : BaseEntity() {
    // ...

    /**
     * 배송지 변경 로직 (비즈니스 규칙 포함)
     */
    fun updateShippingInfo(newAddress: String) {
        // [규칙] 이미 배송 시작된 주문은 배송지를 변경할 수 없다.
        if (this.status != OrderStatus.PENDING) {
            throw IllegalStateException("배송이 시작된 주문은 주소를 변경할 수 없습니다.")
        }
        this.shippingAddress = newAddress
    }

    /**
     * 주문 취소 로직 (비즈니스 규칙 포함)
     */
    fun cancel() {
        // [규칙] 이미 배송 시작된 주문은 취소할 수 없다.
        if (this.status != OrderStatus.PENDING) {
            throw IllegalStateException("배송이 시작된 주문은 취소할 수 없습니다.")
        }
        this.status = OrderStatus.CANCELLED
    }
}

enum class OrderStatus { PENDING, PAID, SHIPPED, DELIVERED, CANCELLED }
```

-----

### 3\. Command Handler 구현 (`OrderService`)

`OrderService`가 Command를 받아 처리하는 **Command Handler**의 역할을 수행합니다. 각 메서드는 Command 객체를 인자로 받아 애그리거트의 상태 변경을 위임합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/service/OrderService.kt
package com.ecommerce.order.service

import com.ecommerce.order.command.*
import com.ecommerce.order.domain.Order
import com.ecommerce.order.domain.OrderRepository
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional // Command는 상태를 변경하므로 클래스 레벨에 @Transactional 적용
class OrderService(
    private val orderRepository: OrderRepository,
    private val productServiceClient: ProductServiceClient, // (06장에서 구현)
) {
    /**
     * 주문 생성 Command 처리
     */
    fun createOrder(command: CreateOrderCommand): Long {
        // (재고 확인 등 외부 서비스 연동 로직...)
        // val product = productServiceClient.getProduct(command.productId)
        
        val newOrder = Order(
            memberId = command.memberId,
            shippingAddress = command.shippingAddress,
            // ...
        )
        val savedOrder = orderRepository.save(newOrder)
        return savedOrder.id!! // 생성된 주문의 ID만 반환
    }

    /**
     * 배송지 변경 Command 처리
     */
    fun updateShippingInfo(command: UpdateShippingInfoCommand) {
        // 1. 애그리거트 루트를 조회
        val order = orderRepository.findByIdOrNull(command.orderId)
            ?: throw EntityNotFoundException("주문을 찾을 수 없습니다.")
        
        // 2. 애그리거트의 비즈니스 메서드 호출하여 상태 변경 위임
        order.updateShippingInfo(command.newShippingAddress)
        
        // 3. 변경 감지(Dirty Checking)에 의해 트랜잭션 커밋 시 UPDATE 쿼리 실행
    }
    
    /**
     * 주문 취소 Command 처리
     */
    fun cancelOrder(command: CancelOrderCommand) {
        val order = orderRepository.findByIdOrNull(command.orderId)
            ?: throw EntityNotFoundException("주문을 찾을 수 없습니다.")
            
        order.cancel()
    }
}
```

-----

### 4\. API 엔드포인트 구현 (`OrderController`)

`OrderController`는 HTTP 요청을 받아 적절한 Command 객체를 생성하고, 이를 `OrderService`(CommandHandler)로 전달(Dispatch)하는 역할만 수행합니다. 컨트롤러에는 비즈니스 로직이 전혀 없습니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/api/OrderController.kt
package com.ecommerce.order.api

import com.ecommerce.order.command.*
import com.ecommerce.order.service.OrderService
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import java.net.URI

@RestController
@RequestMapping("/api/v1/orders")
class OrderController(
    private val orderService: OrderService
) {
    @PostMapping
    fun createOrder(@RequestBody request: CreateOrderRequest): ResponseEntity<Void> {
        // HTTP Request Body를 그대로 Command 객체로 활용
        val command = CreateOrderCommand(
            memberId = 1L, // (인증 헤더에서 가져와야 함)
            productId = request.productId,
            quantity = request.quantity,
            shippingAddress = request.shippingAddress
        )
        
        val orderId = orderService.createOrder(command)
        // 생성된 리소스의 위치를 Location 헤더에 담아 201 Created 응답
        return ResponseEntity.created(URI.create("/api/v1/orders/$orderId")).build()
    }

    @PatchMapping("/{orderId}/shipping-info")
    fun updateShippingInfo(
        @PathVariable orderId: Long,
        @RequestBody request: UpdateShippingInfoRequest
    ): ResponseEntity<Void> {
        val command = UpdateShippingInfoCommand(orderId, request.newAddress)
        orderService.updateShippingInfo(command)
        // 성공적으로 처리되었음을 204 No Content 응답
        return ResponseEntity.noContent().build()
    }

    @PostMapping("/{orderId}/cancel")
    fun cancelOrder(@PathVariable orderId: Long): ResponseEntity<Void> {
        val command = CancelOrderCommand(orderId)
        orderService.cancelOrder(command)
        return ResponseEntity.noContent().build()
    }
}
// (Controller에서 사용할 Request DTO들)
data class CreateOrderRequest(...)
data class UpdateShippingInfoRequest(val newAddress: String)
```

이것으로 '주문'의 상태를 변경하는 **Command 측 파이프라인** 구현이 완료되었습니다.

이제 시스템의 \*\*쓰기 전용 DB(`Write DB`)\*\*에는 최신 상태가 반영되었습니다. 하지만 이 변경 사항을 \*\*읽기 전용 DB(`Read DB`)\*\*는 아직 알지 못합니다. 다음 단계는 사용자의 화면에 보여줄 '조회' 모델을 설계하고, 이 두 모델 간의 데이터를 동기화하는 방법을 구현하는 것입니다.