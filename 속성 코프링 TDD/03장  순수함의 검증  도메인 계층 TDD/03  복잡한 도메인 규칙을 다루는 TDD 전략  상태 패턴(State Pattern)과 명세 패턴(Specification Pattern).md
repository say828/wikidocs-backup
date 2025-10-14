## 03\. 복잡한 도메인 규칙을 다루는 TDD 전략: 상태 패턴(State Pattern)과 명세 패턴(Specification Pattern)

지금까지 우리는 비교적 단순한 비즈니스 규칙(나이 제한, 재고 수량)을 다루었다. 하지만 현실의 도메인은 훨씬 더 복잡하다. 주문의 상태(`결제 대기`, `배송 중`, `주문 취소` 등)에 따라 가능한 행위가 달라지거나, 여러 조건이 복합적으로 얽혀있는 할인 정책처럼 동적으로 변하는 규칙들을 마주하게 된다.

이러한 복잡성을 거대한 `if-else` 블록이나 `when`(switch) 구문으로 처리하려는 시도는 코드를 이해하기 어렵고, 수정하기 두려우며, 테스트하기 불가능한 괴물로 만드는 지름길이다. TDD는 이러한 복잡성에 정면으로 맞설 수 있는 두 가지 강력한 고전 디자인 패턴, \*\*상태 패턴(State Pattern)\*\*과 \*\*명세 패턴(Specification Pattern)\*\*을 자연스럽게 도입하도록 이끈다.

### **상태에 따라 행위가 달라질 때: 상태 패턴**

주문(Order) 객체를 생각해보자. 주문의 상태에 따라 `결제하기`, `배송하기`, `취소하기` 등의 행위가 가능할 수도, 불가능할 수도 있다.

**나쁜 예: `when` 구문으로 상태를 분기하는 코드**

```kotlin
enum class OrderStatus { PENDING, PAID, SHIPPED, CANCELLED }

class Order(var status: OrderStatus) {
    fun ship() {
        when (status) {
            OrderStatus.PAID -> {
                this.status = OrderStatus.SHIPPED
                println("상품을 배송합니다.")
            }
            OrderStatus.PENDING -> throw IllegalStateException("결제되지 않은 주문은 배송할 수 없습니다.")
            OrderStatus.SHIPPED -> throw IllegalStateException("이미 배송된 주문입니다.")
            OrderStatus.CANCELLED -> throw IllegalStateException("취소된 주문은 배송할 수 없습니다.")
        }
    }
    // cancel(), pay() 등 다른 메소드에도 유사한 when 구문이 반복될 것이다...
}
```

이 방식의 문제는 새로운 상태(예: `반품 완료`)가 추가될 때마다 모든 `when` 구문을 찾아 수정해야 한다는 점이다. 이는 OCP(개방-폐쇄 원칙)를 위반하며, 코드를 매우 취약하게 만든다.

**TDD와 상태 패턴의 만남**

"결제 완료된 주문은 배송될 수 있다"는 테스트와 "결제 대기 중인 주문은 배송될 수 없다"는 테스트를 각각 작성하다 보면, 우리는 상태별로 다른 행위를 하는 객체가 필요하다는 사실을 깨닫게 된다. 이것이 바로 상태 패턴의 시작이다.

1.  **상태를 표현하는 인터페이스를 정의한다.**

```kotlin
interface OrderState {
    fun ship(): OrderState
    fun cancel(): OrderState
}
```
    
2.  **각 상태를 별도의 클래스로 구현한다.**

```kotlin
object PaidState : OrderState {
    override fun ship(): OrderState {
        println("상품을 배송합니다.")
        return ShippedState // 다음 상태를 반환
    }
    override fun cancel(): OrderState { /* ... */ }
}

object PendingState : OrderState {
    override fun ship(): OrderState {
        throw IllegalStateException("결제되지 않은 주문은 배송할 수 없습니다.")
    }
    override fun cancel(): OrderState { /* ... */ }
}
// ShippedState, CancelledState 등...
```
    
3.  **Context 객체(Order)는 현재 상태 객체에 행위를 위임한다.**

```kotlin
class Order(private var state: OrderState) { // 이제 Order는 구체적인 status가 아닌, 상태 객체를 가진다.
    fun ship() {
        this.state = this.state.ship() // 행위를 현재 상태 객체에 위임하고, 다음 상태를 받는다.
    }
    fun cancel() {
        this.state = this.state.cancel()
    }
}
```

상태 패턴을 적용하면, 각 상태와 관련된 로직이 해당 상태 클래스 안에 완벽하게 캡슐화된다. 새로운 상태가 추가되더라도 기존의 다른 상태 클래스나 `Order` 클래스를 수정할 필요가 전혀 없다. 각 상태 클래스는 그 자체로 작고 독립적이어서 TDD로 검증하기 매우 용이하다.

### **복잡한 규칙의 조합이 필요할 때: 명세 패턴**

이번에는 "VIP 등급이면서, 최근 한 달 내 구매 금액이 10만 원 이상인 고객에게 특별 할인을 적용한다"와 같은 복잡한 비즈니스 규칙을 다뤄보자.

**나쁜 예: `if` 구문으로 규칙을 조합하는 코드**

```kotlin
fun applySpecialDiscount(customer: Customer) {
    if (customer.grade == Grade.VIP && customer.getPurchaseAmountInLastMonth() >= 100000) {
        // 할인 적용 로직
    }
}
```

이 방식은 규칙이 추가되거나 변경될 때마다 `if` 문의 조건이 계속해서 길어지고 복잡해진다. "VIP이거나, 혹은 VVIP인 경우"와 같이 `OR` 조건이 추가되면 코드는 걷잡을 수 없이 복잡해진다.

**TDD와 명세 패턴의 만남**

명세 패턴은 각각의 비즈니스 규칙을 **재사용 가능한 작은 객체**로 캡슐화하는 기법이다.

1.  **규칙을 표현하는 인터페이스(Specification)를 정의한다.**

```kotlin
interface Specification<T> {
    fun isSatisfiedBy(item: T): Boolean
    fun and(other: Specification<T>): Specification<T>
    fun or(other: Specification<T>): Specification<T>
}
```
    
2.  **개별 규칙을 각각의 클래스로 구현한다.**

```kotlin
class VipCustomerSpecification : Specification<Customer> {
    override fun isSatisfiedBy(customer: Customer) = customer.grade == Grade.VIP
    // ... and, or 구현 ...
}

class RecentPurchaseSpecification(private val amount: Int) : Specification<Customer> {
    override fun isSatisfiedBy(customer: Customer) = customer.getPurchaseAmountInLastMonth() >= amount
    // ... and, or 구현 ...
}
```
    
3.  **규칙들을 TDD로 검증하고, 객체처럼 조합하여 사용한다.**

```kotlin
// TDD로 각 Specification을 철저히 검증한다.

// 검증된 Specification들을 조합하여 새로운 규칙을 만든다.
val specialDiscountRule = VipCustomerSpecification().and(RecentPurchaseSpecification(100000))

if (specialDiscountRule.isSatisfiedBy(someCustomer)) {
    // 할인 적용
}
```

명세 패턴을 사용하면, 복잡한 비즈니스 규칙을 마치 레고 블록처럼 조립하고 재사용할 수 있다. 각각의 규칙은 독립적으로 테스트될 수 있으며, 새로운 규칙이 추가되더라도 기존 코드를 수정할 필요 없이 새로운 명세 클래스를 하나 더 만들면 그만이다.

-----

복잡성은 피할 수 없는 현실이다. 하지만 상태 패턴과 명세 패턴은 TDD와 결합될 때 그 복잡성을 길들이는 가장 우아하고 강력한 방법을 제시한다. TDD는 우리에게 작은 단위로 문제를 나누어 생각하도록 강제하며, 이 두 패턴은 그 나눠진 조각들을 가장 객체지향적인 방식으로 조직화하도록 돕는다. 이것이 바로 TDD가 단순한 코딩 기술을 넘어, 탁월한 설계 도구인 이유다.