## 코틀린 Kotest와 MockK를 활용한 서비스 단위 테스트

테스트 피라미드의 가장 넓고 단단한 기반은 \*\*단위 테스트(Unit Test)\*\*입니다. 단위 테스트의 핵심 철학은 \*\*'격리(Isolation)'\*\*입니다. 즉, 테스트하려는 '단위'(클래스 또는 함수)를 제외한 모든 외부 의존성(DB, 다른 서비스로의 네트워크 호출, 메시지 큐 등)을 **'가짜(Mock)' 객체로 대체**하여, 오직 해당 단위의 순수한 비즈니스 로직만을 정밀하게 검증하는 것입니다.

단위 테스트는 실행 속도가 수 밀리초(ms) 단위로 매우 빠르기 때문에, 개발자는 코드 변경 시 수백, 수천 개의 테스트를 몇 초 안에 실행하여 자신의 변경이 다른 곳에 미치는 영향을 즉시 피드백받을 수 있습니다.

03장에서 우리는 이미 `Kotest`와 `MockK`를 경험했지만, 이 절에서는 MSA의 복잡한 상호작용을 테스트하는 구체적인 시나리오를 통해 이 도구들의 진정한 힘을 알아보겠습니다.

-----

### 왜 Kotest와 MockK인가?

  * **Kotest:** `Given-When-Then`(BehaviorSpec)과 같이 테스트의 '행위'를 서술적으로 작성할 수 있게 하여, 테스트 코드가 그 자체로 '살아있는 문서'가 되게 합니다. `shouldBe`, `shouldThrow` 등 직관적인 Matcher는 테스트 코드의 가독성을 극대화합니다.
  * **MockK:** `Mockito`와 같은 전통적인 자바 Mock 라이브러리들이 코틀린의 `final` 클래스나 `suspend` 함수, 확장 함수 등을 다루기 까다로운 반면, **MockK는 처음부터 코틀린을 위해 설계**되어 이러한 언어적 특성을 완벽하고 간결하게 지원하는 최고의 Mocking 프레임워크입니다.

-----

### 실전 예제: `OrderService` 단위 테스트

'주문 생성' 로직을 담고 있는 `OrderService`를 단위 테스트해 봅시다. `OrderService`는 다음과 같은 외부 의존성을 가집니다.

  * `OrderRepository`: 데이터베이스
  * `ProductServiceClient`: `product-service`로의 Feign 호출
  * `OrderEventProducer`: Kafka로의 이벤트 발행

단위 테스트에서는 이 3가지 의존성을 모두 `MockK`를 이용해 가짜 객체로 만듭니다.

```kotlin
// order-service/src/test/kotlin/com/ecommerce/order/service/OrderServiceTest.kt
package com.ecommerce.order.service

import com.ecommerce.order.client.ProductResponse
import com.ecommerce.order.client.ProductServiceClient
import com.ecommerce.order.domain.Order
import com.ecommerce.order.domain.OrderRepository
import io.kotest.assertions.throwables.shouldThrow
import io.kotest.core.spec.style.BehaviorSpec
import io.kotest.matchers.shouldBe
import io.mockk.*

class OrderServiceTest : BehaviorSpec({
    // 1. 모든 의존성을 mockk<>()를 통해 Mock 객체로 생성
    val orderRepository = mockk<OrderRepository>()
    val productServiceClient = mockk<ProductServiceClient>()
    val orderEventProducer = mockk<OrderEventProducer>()

    // 2. 테스트 대상(SUT)에 Mock 객체들을 주입
    val orderService = OrderService(orderRepository, productServiceClient, orderEventProducer)

    // 3. 각 테스트 케이스 실행 후 Mock 객체의 모든 기록을 초기화
    afterTest {
        clearMocks(orderRepository, productServiceClient, orderEventProducer)
    }

    // --- 테스트 케이스 1: 주문 생성 성공 ---
    Given("상품 재고가 충분하고 모든 조건이 정상일 때") {
        val memberId = 1L
        val productId = 100L
        val quantity = 2

        // 4. Mock 객체의 행동 정의 (every ~ returns)
        // 4-1. 상품 서비스는 충분한 재고를 가진 상품 정보를 반환한다
        every { productServiceClient.getProduct(productId) } returns ProductResponse(
            id = productId, name = "Test Product", price = 10000L, stockQuantity = 10
        )
        // 4-2. 주문 저장소는 저장된 Order 객체를 반환한다 (slot을 이용해 저장되는 객체 캡처)
        val savedOrderSlot = slot<Order>()
        every { orderRepository.save(capture(savedOrderSlot)) } returnsArgument 0
        // 4-3. 이벤트 프로듀서는 아무것도 하지 않는다 (Unit 반환)
        every { orderEventProducer.sendOrderCreatedEvent(any()) } just Runs

        When("주문 생성을 요청하면") {
            val createdOrder = orderService.createOrder(memberId, productId, quantity)

            Then("주문이 성공적으로 생성되고 저장되어야 한다") {
                // 5. 특정 메서드가 호출되었는지 검증 (verify)
                verify(exactly = 1) { orderRepository.save(any()) }
                createdOrder.totalPrice shouldBe 20000L // (10000 * 2)

                // 캡처된 Order 객체의 상태를 검증
                savedOrderSlot.captured.status shouldBe OrderStatus.PENDING
            }
            
            Then("주문 생성 이벤트가 발행되어야 한다") {
                verify(exactly = 1) { orderEventProducer.sendOrderCreatedEvent(any()) }
            }
        }
    }

    // --- 테스트 케이스 2: 재고 부족으로 주문 생성 실패 ---
    Given("상품 재고가 주문 수량보다 부족할 때") {
        val memberId = 1L
        val productId = 101L
        val quantity = 5

        // 상품 서비스는 재고가 3개뿐인 상품 정보를 반환한다
        every { productServiceClient.getProduct(productId) } returns ProductResponse(
            id = productId, name = "Low Stock Product", price = 5000L, stockQuantity = 3
        )

        When("주문 생성을 요청하면") {
            
            Then("IllegalArgumentException 예외가 발생해야 한다") {
                // 6. 특정 예외가 발생하는지 검증 (shouldThrow)
                val exception = shouldThrow<IllegalArgumentException> {
                    orderService.createOrder(memberId, productId, quantity)
                }
                exception.message shouldBe "상품 재고가 부족합니다."
            }

            Then("주문은 절대 저장되거나 이벤트가 발행되지 않아야 한다") {
                verify(exactly = 0) { orderRepository.save(any()) }
                verify(exactly = 0) { orderEventProducer.sendOrderCreatedEvent(any()) }
            }
        }
    }
})
```

  * **`suspend` 함수 테스트:** 만약 `productServiceClient.getProduct`이 `suspend` 함수였다면, `every` 대신 `coEvery`를, `verify` 대신 `coVerify`를 사용하여 동일하게 테스트할 수 있습니다. `MockK`는 코루틴 테스트를 완벽하게 지원합니다.

-----

단위 테스트는 우리 테스트 전략의 가장 중요한 토대입니다. `Kotest`의 서술적인 테스트 구조와 `MockK`의 강력한 Mocking 기능을 통해, 우리는 복잡한 MSA 환경에서도 각 서비스의 비즈니스 로직을 빠르고 안정적으로 검증하여 CI 파이프라인의 속도와 안정성을 보장할 수 있습니다.