# 03장: 순수함의 검증: 도메인 계층 TDD

2장에서 우리는 코틀린의 표현력을 극대화하는 강력한 테스트 도구인 Kotest와 MockK를 손에 넣었다. 이제 이론과 도구라는 양날의 검을 들고, 실제 소프트웨어 아키텍처의 가장 중요한 심장부를 향해 진격할 시간이다. 그곳은 바로 \*\*도메인 계층(Domain Layer)\*\*이다.

애플리케이션을 하나의 유기체에 비유한다면, 도메인 계층은 뇌와 심장이다. 시스템이 '무엇을 하는지', '어떤 가치를 제공하는지'에 대한 모든 본질적인 지식과 규칙이 이곳에 담겨있다. 프레임워크, 데이터베이스, UI와 같은 다른 모든 기술적 요소들은 이 심장이 계속해서 뛸 수 있도록 영양분을 공급하고 외부와 상호작용하는 팔다리에 불과하다.

우리가 TDD 여정을 도메인 계층에서 시작하는 이유는 명확하다. 첫째, 이곳이 시스템에서 가장 **중요하고 변하지 않는 가치**를 담고 있기 때문이다. 둘째, 도메인 계층은 데이터베이스나 웹 프레임워크 같은 외부 기술로부터 완벽히 독립된 **'순수한' 코드**여야 하므로, 우리가 1장에서 배운 빠르고(Fast) 독립적인(Independent) 단위 테스트를 적용하기에 가장 이상적인 공간이다. 견고한 도메인 없이는 신뢰할 수 있는 소프트웨어도 없다. 이 장에서는 TDD를 통해 깨끗하고 강력한 도메인 모델을 조각해 나가는 과정을 A부터 Z까지 학습할 것이다.

-----

## 00\. 도메인 모델과 비즈니스 로직: 시스템의 핵심 가치를 지키는 첫 번째 방어선

소프트웨어 개발의 세계에는 오랫동안 이어진 잘못된 관행이 있다. 바로 \*\*'빈혈증 도메인 모델(Anemic Domain Model)'\*\*이다. 이는 도메인 객체가 아무런 비즈니스 로직 없이 단순히 데이터(상태)를 담는 컨테이너 역할만 하고, 관련된 모든 처리 로직은 별도의 서비스(Service) 클래스에 위임하는 안티패턴을 말한다.

**나쁜 예: 빈혈증 도메인 모델**

```kotlin
// 데이터만 가진 깡통 클래스
class AnemicOrder {
    var id: Long? = null
    var status: OrderStatus = OrderStatus.PENDING
    var orderLines: List<OrderLine> = emptyList()
}

// 모든 비즈니스 로직이 외부에 흩어져 있다.
class AnemicOrderService {
    fun ship(order: AnemicOrder) {
        if (order.status != OrderStatus.PAID) {
            throw IllegalStateException("결제되지 않은 주문은 배송할 수 없습니다.")
        }
        order.status = OrderStatus.SHIPPED
        // ...
    }
}
```

이러한 설계는 객체지향의 핵심인 \*\*'데이터와 행동을 함께 캡슐화'\*\*하는 원칙을 위배한다. `Order`와 관련된 규칙(결제되어야 배송 가능)이 `Order` 자기 자신이 아닌 `OrderService`에 있기 때문에, `Order`의 상태는 외부 어디에서든 쉽게 오염될 수 있다.

TDD는 이러한 빈혈증 모델을 피하고, 데이터와 행위가 응집된 \*\*'풍부한 도메인 모델(Rich Domain Model)'\*\*을 만들도록 자연스럽게 유도한다.

**좋은 예: TDD를 통해 탄생한 풍부한 도메인 모델**

우리가 "결제 완료되지 않은 주문을 배송하면 예외가 발생한다"는 테스트를 먼저 작성한다고 상상해보자.

```kotlin
// 실패하는 테스트 먼저 작성
@Test
fun `결제되지 않은 주문을 배송하려고 하면 예외가 발생해야 한다`() {
    val unpaidOrder = Order(status = OrderStatus.PENDING)
    
    shouldThrow<IllegalStateException> {
        unpaidOrder.ship() // ship() 이라는 행위가 Order 내부에 있기를 기대하게 된다.
    }
}
```

이 테스트를 통과시키기 위한 가장 자연스러운 방법은, `ship()`이라는 행위를 `Order` 클래스 내부로 가져오는 것이다.

```kotlin
// 테스트를 통과시키는 프로덕션 코드
class Order(
    private var status: OrderStatus
) {
    fun ship() {
        // '배송'이라는 행위의 사전 조건(pre-condition)을 스스로 검증한다.
        if (this.status != OrderStatus.PAID) {
            throw IllegalStateException("결제되지 않은 주문은 배송할 수 없습니다.")
        }
        this.status = OrderStatus.SHIPPED
    }
}
```

이제 `Order` 객체는 더 이상 수동적인 데이터 덩어리가 아니다. 스스로의 상태(status)를 보호하고, 비즈니스 규칙(결제 완료)을 강제하는 능동적인 주체가 되었다. 이것이 바로 풍부한 도메인 모델이다.

**도메인 모델은 시스템의 '첫 번째 방어선'이다.** 잘못된 데이터나 비즈니스 규칙에 위배되는 상태 변경 시도를 가장 먼저, 가장 안쪽에서 차단하는 역할을 수행해야 한다. 예를 들어, 상품의 재고(stock)를 관리하는 `Product` 객체가 있다면, 재고가 음수가 되는 것을 허용해서는 안 된다.

```kotlin
class Product(
    var stock: Long
) {
    fun decreaseStock(quantity: Long) {
        if (this.stock < quantity) {
            throw IllegalArgumentException("재고가 충분하지 않습니다.")
        }
        this.stock -= quantity
    }
}
```

`decreaseStock` 메소드는 재고가 음수가 될 가능성을 원천적으로 차단한다. 이처럼 견고하게 만들어진 도메인 모델이 있다면, 이 모델을 사용하는 상위 계층(애플리케이션 서비스, 컨트롤러 등)은 훨씬 더 안심하고 비즈니스 흐름에만 집중할 수 있다.

결론적으로, 도메인 계층에서의 TDD는 단순히 버그를 찾는 행위를 넘어선다. 그것은 도메인 전문가와 소통하며 얻은 비즈니스 지식을 **가장 순수하고, 가장 견고하며, 가장 표현력 있는 코드**로 응축시키는 설계 과정이다. 이제부터 우리는 TDD를 통해 이러한 살아있는 도메인 모델을 만들어가는 구체적인 기술들을 탐험할 것이다.