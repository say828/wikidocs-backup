## 확장 함수(Extension Functions)를 활용한 비즈니스 로직과 인프라 로직의 분리

우리는 02장(DDD)에서 '순수한 도메인 모델'을 설계하는 것을 목표로 합니다. '순수한 도메인 모델'이란, 자신의 핵심 비즈니스 규칙(예: '주문은 총액이 0원 미만일 수 없다')에만 집중하고, 다른 기술적 관심사(예: '이 객체를 JSON DTO로 어떻게 변환할 것인가?')에는 전혀 관심이 없는 객체를 의미합니다.

하지만 스프링 부트 애플리케이션을 개발하다 보면 이 '순수성'을 지키기 어려운 유혹에 빠집니다.

가령, `Order`라는 JPA `@Entity`가 있습니다. 서비스 계층에서 이 `Order`를 조회한 뒤 컨트롤러로 반환하기 위해 `OrderResponse`라는 DTO로 변환해야 합니다.

**방법 1: 엔티티 내부에 변환 메서드 구현 (나쁜 방법)**

```kotlin
@Entity
class Order(
    // ... Order의 순수 속성들
    var status: OrderStatus
) {
    // ... Order의 핵심 비즈니스 로직
    fun ship() { this.status = OrderStatus.SHIPPED }

    // ⛔️ 도메인 순수성 위반! ⛔️
    // 'Order'가 'OrderResponse'라는 외부 계층(Web/DTO)을 알아버렸습니다.
    fun toResponse(): OrderResponse {
        return OrderResponse(
            orderId = this.id,
            status = this.status.name
            // ...
        )
    }
}
```

이 방식은 최악입니다. 도메인 계층이 애플리케이션/웹 계층에 직접 의존하게 되면서 계층 간의 경계가 무너지고, `Order` 엔티티는 온갖 DTO 변환 코드로 오염됩니다.

**방법 2: 별도의 변환기(Converter) 클래스 구현 (전통적인 방법)**

```kotlin
// 1. Order 엔티티는 순수성을 유지
@Entity
class Order(...) { ... }

// 2. 별도의 변환기 클래스
@Component
class OrderConverter {
    fun toResponse(order: Order): OrderResponse {
        return OrderResponse(
            orderId = order.id,
            status = order.status.name
            // ...
        )
    }
}

// 3. 서비스 계층에서 사용
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val orderConverter: OrderConverter // 👎 변환기 의존성 추가
) {
    fun getOrder(orderId: Long): OrderResponse {
        val order = orderRepository.findByIdOrNull(orderId) 
            ?: throw EntityNotFoundException()
        
        // 👎 문법이 장황하고, 객체지향적이지 않음
        return orderConverter.toResponse(order) 
    }
}
```

이 방식은 계층 분리에는 성공했지만, `OrderConverter`라는 불필요한 클래스가 늘어나고 `OrderService`는 `OrderConverter`에 의존해야 합니다. 또한 `order.toResponse()`라는 직관적인 객체지향적 호출이 불가능합니다.

-----

### 코틀린의 해법: 확장 함수 (Extension Functions)

코틀린의 **확장 함수**는 이 문제를 가장 우아하게 해결합니다. 확장 함수는 **클래스의 원본 코드를 수정하지 않으면서** 마치 그 클래스의 멤버 함수인 것처럼 새로운 함수를 '추가'할 수 있게 해줍니다.

실제로는 정적(static) 유틸리티 메서드처럼 동작하지만, 문법적으로는 멤버 함수처럼 `.`(점)을 찍어 호출할 수 있습니다.

`OrderConverter` 클래스 대신, `OrderDtoExtensions.kt` 같은 별도 파일을 만들고 다음과 같이 작성합니다.

```kotlin
// file: OrderDtoExtensions.kt
// (특정 클래스에 속하지 않는, 최상위 레벨 함수로 선언)

/**
 * 'Order' 엔티티를 'OrderResponse' DTO로 변환하는 확장 함수
 * * fun [수신 객체 타입].[함수명](): [반환 타입]
 */
fun Order.toResponse(): OrderResponse {
    // 'this'는 Order 객체 자신을 가리킵니다.
    return OrderResponse(
        orderId = this.id!!, // 이 시점엔 ID가 있음을 보장
        memberId = this.memberId,
        status = this.status.name,
        // ...
    )
}

// 요청 DTO를 엔티티로 변환하는 확장 함수도 가능합니다.
fun CreateOrderRequest.toEntity(memberId: Long): Order {
    return Order(
        memberId = memberId,
        status = OrderStatus.PENDING,
        // ...
    )
}
```

이제 `OrderService`는 `OrderConverter`에 의존할 필요가 없습니다.

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository
    // 👍 변환기 의존성(OrderConverter)이 사라짐!
) {
    fun getOrder(orderId: Long): OrderResponse {
        val order = orderRepository.findByIdOrNull(orderId) 
            ?: throw EntityNotFoundException()
        
        // 👍 마치 Order의 원래 멤버 함수인 것처럼 호출 가능!
        return order.toResponse() 
    }

    fun createOrder(memberId: Long, request: CreateOrderRequest): OrderResponse {
        // 👍 DTO -> Entity 변환도 깔끔하게 처리
        val newOrder = request.toEntity(memberId)
        val savedOrder = orderRepository.save(newOrder)
        return savedOrder.toResponse()
    }
}
```

### 확장 함수의 장점 요약

1.  **도메인 순수성 보장:** `Order` 엔티티는 `OrderResponse` DTO의 존재를 전혀 알 필요가 없습니다. 비즈니스 로직(Domain)과 인프라/웹 로직(DTO)이 완벽하게 분리됩니다.
2.  **가독성 향상:** `orderConverter.toResponse(order)`보다 `order.toResponse()`가 훨씬 직관적이고 객체지향적입니다.
3.  **유틸리티성 로직 분리:** 꼭 DTO 변환이 아니더라도, `Product` 엔티티에 `fun Product.isOutOfStock(): Boolean` 같은 편의성 검증 로직을 '확장'하여 도메인 모델의 핵심 책임과 부가적인 책임을 분리할 수 있습니다.

이처럼 확장 함수는 스프링 부트 애플리케이션의 '관심사의 분리(Separation of Concerns)'를 코드 레벨에서 실현하는 강력한 도구이며, 우리가 02장에서 다룰 DDD의 사상과도 완벽하게 일치합니다.