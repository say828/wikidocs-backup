## 02\. 불변 객체(Immutable Object)와 테스트: 예측 가능하고 안정적인 도메인 모델 구축

우리는 테스트 케이스 설계를 통해 도메인 모델의 규칙을 견고하게 만드는 법을 배웠다. 하지만 비즈니스 규칙을 강제하는 것만으로는 충분하지 않다. 우리가 만든 객체가 시스템의 복잡한 흐름 속에서 **예측 가능하게** 동작하도록 보장해야 한다. 바로 이 지점에서 객체지향 설계의 가장 강력한 개념 중 하나인 \*\*불변성(Immutability)\*\*이 등장한다.

**불변 객체란, 생성된 이후에는 그 내부 상태를 절대 변경할 수 없는 객체**를 말한다. `String` 클래스를 생각해보자. `toUpperCase()` 메소드를 호출해도 원본 문자열은 바뀌지 않고, 대문자로 변환된 새로운 `String` 객체가 반환된다. 이것이 바로 불변성의 핵심이다.

왜 도메인 모델을 불변 객체로 설계하는 것이 중요할까?

1.  **예측 가능성과 안정성:** 객체의 상태가 변하지 않으므로, 그 객체를 여러 곳에서 참조하더라도 언제나 동일한 값을 가지고 있음을 보장할 수 있다. 이는 복잡한 시스템에서 "대체 이 값이 어디서 바뀐 거지?"라며 몇 시간씩 디버깅하는 끔찍한 상황을 원천적으로 차단한다.
2.  **동시성 문제 해결:** 멀티스레드 환경에서 가장 큰 골칫거리는 여러 스레드가 공유된 데이터(객체)를 동시에 수정하려 할 때 발생하는 경쟁 상태(Race Condition)다. 객체가 불변이라면 '수정' 자체가 불가능하므로, 락(Lock)과 같은 복잡한 동기화 메커니즘 없이도 스레드로부터 안전(Thread-safe)하다.
3.  **테스트의 단순화:** 테스트에서 불변 객체의 가장 큰 장점이다. 객체의 상태가 고정되어 있으므로, 우리는 어떤 연산을 수행하기 전과 후의 상태를 걱정할 필요가 없다. 오직 '입력'과 '출력(새로운 객체)'에만 집중하면 된다. 테스트의 준비(Setup)와 검증(Assertion)이 극도로 단순해진다.

### **가변 객체가 초래하는 테스트의 어려움**

가변(Mutable) 객체는 테스트를 얼마나 어렵게 만드는지 직접 확인해보자. 주문 항목의 가격을 나타내는 `Money` 클래스가 가변이라고 가정하자.

**나쁜 예: 가변 `Money` 클래스와 불안정한 테스트**

```kotlin
// 상태 변경이 가능한 가변 클래스
class MutableMoney(var amount: BigDecimal) {
    fun add(other: MutableMoney) {
        this.amount = this.amount.add(other.amount) // 자신의 상태를 직접 변경한다.
    }
}

@Test
fun `가변 객체는 테스트를 오염시킨다`() {
    val price1 = MutableMoney(BigDecimal(1000))
    val price2 = MutableMoney(BigDecimal(2000))
    val cart = ShoppingCart(price1, price2) // 카트는 두 가격 객체를 참조한다.

    // 카트의 총액을 계산한다.
    val total = cart.calculateTotal()
    total.amount shouldBe BigDecimal(3000)

    // 이런! 다른 곳에서 price1 객체의 상태를 변경해버렸다!
    price1.add(MutableMoney(BigDecimal(500))) // price1의 amount는 이제 1500

    // 동일한 cart 객체로 다시 총액을 계산하면?
    val corruptedTotal = cart.calculateTotal()
    
    // 테스트는 실패한다! 3000을 기대했지만 3500이 나온다.
    // cart 객체는 아무런 변경이 없었음에도 불구하고 결과가 달라졌다.
    corruptedTotal.amount shouldBe BigDecimal(3000) 
}
```

이처럼 가변 객체는 보이지 않는 부수 효과(Side Effect)를 만들어내며, 테스트를 예측 불가능하고 신뢰할 수 없는 상태로 만든다.

### **TDD를 통한 불변 객체 설계**

이제 "두 `Money` 객체를 더하면 새로운 `Money` 객체가 반환된다"는 실패하는 테스트를 먼저 작성해보자. 이 테스트는 자연스럽게 우리를 불변 설계로 이끈다.

**좋은 예: TDD로 설계한 불변 `Money` 클래스**

```kotlin
// 1. 실패하는 테스트 먼저 작성 (RED)
@Test
fun `두 Money 객체를 더하면, 두 금액의 합을 가진 새로운 Money 객체가 반환되어야 한다`() {
    val moneyA = Money(BigDecimal(1000))
    val moneyB = Money(BigDecimal(2000))

    val result = moneyA.add(moneyB)

    // 결과는 새로운 객체여야 한다.
    result shouldNotBeSameInstanceAs moneyA
    result shouldNotBeSameInstanceAs moneyB
    result.amount shouldBe BigDecimal(3000)

    // 원본 객체들은 절대 변해서는 안 된다.
    moneyA.amount shouldBe BigDecimal(1000)
    moneyB.amount shouldBe BigDecimal(2000)
}

// 2. 테스트를 통과시키는 코드 작성 (GREEN & REFACTOR)
// 코틀린의 data class와 val 키워드는 불변 객체를 만들기 위한 최고의 도구다.
data class Money(val amount: BigDecimal) { // amount는 val이므로 재할당 불가
    fun add(other: Money): Money {
        // 자신의 상태를 바꾸지 않고, 새로운 객체를 생성하여 반환한다.
        return Money(this.amount.add(other.amount))
    }
}
```

코틀린의 `data class`는 프로퍼티를 `val`로 선언하는 것만으로 손쉽게 불변 객체를 만들 수 있도록 도와준다. 이 불변 `Money` 객체는 언제 어디서 사용되더라도 그 상태가 변할 걱정이 없으므로, 시스템 전체의 안정성과 예측 가능성을 크게 높여준다.

도메인 모델을 설계할 때, 우리는 항상 스스로에게 질문해야 한다. **"이 객체는 정말로 상태가 변해야만 하는가?"** 대부분의 경우, 그 답은 '아니오'다. TDD를 통해 도메인 객체를 불변으로 설계하는 습관은, 테스트하기 쉬울 뿐만 아니라 근본적으로 더 안정적이고 이해하기 쉬운 코드를 만드는 지름길이다.