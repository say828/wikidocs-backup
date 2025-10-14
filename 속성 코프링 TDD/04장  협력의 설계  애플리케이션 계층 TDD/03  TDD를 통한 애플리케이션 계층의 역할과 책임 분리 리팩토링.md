## 03\. TDD를 통한 애플리케이션 계층의 역할과 책임 분리 리팩토링

TDD의 `Red-Green-Refactor` 사이클에서 가장 간과되기 쉽지만, 가장 심오한 단계는 바로 \*\*리팩토링(Refactor)\*\*이다. 애플리케이션 계층에서 리팩토링은 단순히 코드를 정리하는 것을 넘어, 객체들의 역할과 책임을 재조정하고 아키텍처 전체의 건강성을 증진시키는 **설계 개선 활동**이다.

TDD를 하다 보면 종종 "이 테스트는 작성하기가 너무 어렵다"고 느껴지는 순간이 온다. 초심자는 테스트 코드나 자신의 능력을 탓하지만, 전문가는 그 '어려움'을 \*\*설계의 문제점을 드러내는 냄새(Design Smell)\*\*로 받아들인다. 테스트는 우리 코드의 첫 번째 사용자로서, 코드의 설계가 얼마나 유연하고 합리적인지를 알려주는 가장 정직한 거울이다.

### **설계 냄새 1: 거대한 서비스 클래스 (The God Service)**

  * **증상:** 하나의 애플리케이션 서비스(`UserService`)가 회원 가입, 탈퇴, 정보 수정, 비밀번호 변경, 프로필 조회 등 수많은 관련 유스케이스를 전부 처리한다.
  * **TDD가 보내는 신호:** 테스트 클래스가 비대해진다. `beforeTest` 블록에서 수많은 종류의 리포지토리와 클라이언트를 `mockk`으로 만들어야 한다. 특정 테스트는 그중 두세 개만 사용함에도 불구하고 말이다. 단일 테스트 파일에 수십 개의 `given` 블록이 생겨나고, 원하는 테스트를 찾기조차 어려워진다.
  * **리팩토링 처방:** \*\*단일 책임 원칙(SRP)\*\*을 적용하여 거대 서비스를 유스케이스 단위로 분리한다. `UserService`를 `RegisterUserService`, `DeactivateUserService`, `ChangePasswordService` 등으로 나누는 것이다.

**리팩토링 전:**

```kotlin
class UserService(
    private val userRepository: UserRepository,
    private val emailClient: EmailClient,
    private val passwordEncoder: PasswordEncoder,
    private val auditLogger: AuditLogger
) {
    fun register(...) { /* ... */ }
    fun deactivate(...) { /* ... */ }
    fun changePassword(...) { /* ... */ }
}
```

**리팩토링 후:**

```kotlin
class RegisterUserService(
    private val userRepository: UserRepository,
    private val emailClient: EmailClient,
    private val passwordEncoder: PasswordEncoder
) { /* ... */ }

class DeactivateUserService(
    private val userRepository: UserRepository,
    private val emailClient: EmailClient,
    private val auditLogger: AuditLogger
) { /* ... */ }
```

이제 각 서비스는 자신에게 필요한 의존성만 가지게 되어 테스트 작성이 훨씬 간결해진다. 시스템의 구조 또한 유스케이스 단위로 명확하게 드러난다.

### **설계 냄새 2: 서비스에 스며든 도메인 로직**

  * **증상:** 애플리케이션 서비스가 도메인 객체의 데이터를 꺼내와(get) `if-else`로 비즈니스 규칙을 처리하고, 다시 도메인 객체에 데이터를 집어넣는다(set).
  * **TDD가 보내는 신호:** 테스트의 `when`이나 `then` 블록이 복잡해진다. 서비스의 행위를 검증하기 위해 여러 의존 객체(Stub)로부터 데이터를 가져오고, 테스트 코드 내에서 비즈니스 로직을 재현하며, 최종적으로 `save` 메소드에 전달되는 객체의 상태를 검증하기 위해 복잡한 `capture`와 단언문이 필요해진다.
  * **리팩토링 처방:** **도메인 모델에 책임을 위임한다.** 서비스에 흩어져 있는 비즈니스 로직을 가장 적절한 도메인 객체 내부로 옮겨 캡슐화한다.

**리팩토링 전:**

```kotlin
// In Application Service
fun placeOrder(orderId: Long, productId: String, quantity: Int) {
    val order = orderRepository.findById(orderId)
    val product = productRepository.findById(productId)

    if (product.stock < quantity) { // 비즈니스 로직이 서비스에 있다!
        throw OutOfStockException()
    }

    order.addOrderLine(product, quantity)
    orderRepository.save(order)
}
```

**리팩토링 후:**

```kotlin
// In Domain Model (Order.kt)
class Order(...) {
    fun addProduct(product: Product, quantity: Int) {
        if (product.stock < quantity) { // 비즈니스 로직이 도메인 모델 안에 있다!
            throw OutOfStockException()
        }
        this.orderLines.add(OrderLine(product.id, quantity))
    }
}

// In Application Service
fun placeOrder(orderId: Long, productId: String, quantity: Int) {
    val order = orderRepository.findById(orderId)
    val product = productRepository.findById(productId)

    order.addProduct(product, quantity) // 서비스는 단순히 도메인에 위임한다.
    orderRepository.save(order)
}
```

이제 `Order` 클래스에 대한 단위 테스트를 통해 재고 확인 로직을 견고하게 검증할 수 있으며, `OrderService` 테스트는 `order.addProduct`가 올바르게 호출되었는지만을 검증하면 되므로 매우 단순해진다.

-----

TDD에서 리팩토링 단계는 선택이 아닌 필수다. 테스트는 우리의 설계에 대한 끊임없는 피드백을 제공하는 스파링 파트너와 같다. 테스트 작성이 고통스럽다면, 그것은 테스트가 잘못된 것이 아니라 우리의 코드가 "더 나은 설계를 원한다"고 비명을 지르는 것이다. 이 신호에 귀를 기울이고 과감하게 코드를 개선해 나갈 때, 우리는 비로소 TDD를 통해 진정으로 유연하고 유지보수하기 좋은 아키텍처를 손에 넣을 수 있다.

이것으로 애플리케이션 계층에서의 TDD 여정을 마친다. 우리는 도메인이라는 배우와 그들을 지휘하는 서비스라는 감독을 모두 TDD로 설계하고 검증했다. 이제 남은 것은 이 모든 것이 실제로 동작할 수 있도록 무대 뒤에서 묵묵히 일하는 스태프, 즉 **인프라스트럭처 계층**을 신뢰할 수 있게 만드는 것이다. 다음 장에서는 외부 세계와의 접점을 테스트하는 실전 전략을 다룰 것이다.