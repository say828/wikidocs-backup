## 02\. 고전파(Classicist) vs 런던파(London School) TDD: 상태 기반 검증과 행위 기반 검증

TDD의 세계에는 오랫동안 이어진 두 개의 주요 학파가 존재한다. 바로 \*\*고전파(Classicist 또는 Detroit School)\*\*와 \*\*런던파(London School 또는 Mockist School)\*\*다. 이 둘의 차이를 이해하는 것은 우리가 어떤 상황에서 어떤 방식으로 테스트를 설계해야 하는지 결정하는 데 깊은 통찰을 준다. 이 논쟁은 '누가 옳은가'의 문제가 아니라, '무엇을 검증하고 싶은가'라는 근본적인 질문에 대한 철학의 차이다.

### **고전파 TDD (Classicist TDD): 상태 기반 검증**

고전파 TDD는 테스트하려는 대상(SUT, System Under Test)을 그와 협력하는 실제 객체들과 함께 하나의 단위로 간주한다. 테스트 더블(Test Double), 특히 목(Mock) 객체의 사용을 최소화하며, 협력 객체가 실제와 너무 다르거나(예: 네트워크 통신), 너무 느릴 때(예: 데이터베이스)에만 제한적으로 사용한다.

  * **핵심 철학:** "어떤 연산을 수행한 후, 시스템의 \*\*상태(State)\*\*가 기대하는 대로 변경되었는가?"
  * **검증 방식:** **상태 기반 검증 (State-based Verification)**. 메소드를 호출한 뒤, 그 결과로 반환된 값이나 객체의 특정 프로퍼티 값이 예상과 일치하는지를 단언(assert)한다.

**예시: 고전파 스타일의 테스트**

```kotlin
@Test
fun `상품을 주문하면 재고가 감소해야 한다`() {
    // given: 실제 Product 객체와 OrderService 객체를 생성한다.
    val product = Product(id = "P001", name = "TDD 티셔츠", stock = 10)
    val productService = ConcreteProductService(productRepository = InMemoryProductRepository(product))
    val orderService = OrderService(productService)
    
    // when: 주문 로직을 실행한다.
    orderService.placeOrder(productId = "P001", quantity = 2)
    
    // then: '상태'를 직접 검증한다. Product 객체의 재고(stock)가 정말 8로 바뀌었는지 확인한다.
    val updatedProduct = productService.getProduct("P001")
    assertEquals(8, updatedProduct.stock)
}
```

고전파 스타일은 내부 구현의 세부 사항보다는 최종 결과에 집중하므로, 객체 간의 상호작용 방식을 리팩토링하더라도 테스트가 깨질 가능성이 적다는 장점이 있다.

-----

### **런던파 TDD (London School TDD): 행위 기반 검증**

런던파 TDD는 테스트 대상을 철저하게 고립시키는 것을 원칙으로 한다. 테스트 대상이 의존하는 모든 협력 객체는 목(Mock) 객체로 대체된다. 이 접근법은 테스트가 오직 하나의 클래스, 단 하나의 책임에만 집중하도록 강제한다.

  * **핵심 철학:** "어떤 연산을 수행했을 때, 우리 객체가 협력 객체들에게 올바른 \*\*메시지(Method Call)\*\*를 보냈는가?"
  * **검증 방식:** **행위 기반 검증 (Behavior-based Verification)**. 메소드를 호출한 뒤, 의존하는 목 객체의 특정 메소드가 예상된 인자와 함께 정확한 횟수만큼 호출되었는지를 검증(verify)한다.

**예시: 런던파 스타일의 테스트**

```kotlin
@Test
fun `상품을 주문하면 ProductService의 재고 감소 메소드를 호출해야 한다`() {
    // given: 협력 객체인 ProductService를 Mock으로 만든다.
    val mockProductService = mockk<ProductService>(relaxed = true)
    val orderService = OrderService(mockProductService)
    
    // when: 주문 로직을 실행한다.
    orderService.placeOrder(productId = "P001", quantity = 2)
    
    // then: '행위'를 검증한다. OrderService가 mockProductService의 decreaseStock 메소드를
    // productId="P001", quantity=2 인자와 함께 정확히 1번 호출했는지 확인한다.
    verify(exactly = 1) {
        mockProductService.decreaseStock(productId = "P001", quantity = 2)
    }
}
```

런던파 스타일은 테스트 실패 시 원인이 되는 클래스를 즉시 특정할 수 있으며, 객체 간의 책임과 역할을 명확히 정의하도록 유도하여 자연스럽게 의존성이 낮은 설계를 이끌어낸다는 장점이 있다.

-----

### **어느 길을 선택할 것인가? 전문가의 실용적 접근법**

[도표: 상태 기반 검증과 행위 기반 검증의 비교]

| 특징 | 고전파 (상태 기반) | 런던파 (행위 기반) |
| :--- | :--- | :--- |
| **단위의 범위** | SUT + 실제 협력 객체 | SUT (하나의 클래스) |
| **검증 대상** | 연산 후의 최종 **상태** | 협력 객체와의 **상호작용** (메소드 호출) |
| **리팩토링 유연성**| 높음 (내부 구현 변경에 강함) | 낮음 (상호작용 변경 시 테스트 깨짐) |
| **실패 원인 파악**| 상대적으로 어려움 | 즉각적이고 명확함 |
| **주요 사용처** | 도메인 로직, 값 객체 | 컨트롤러, 서비스 등 협력 조정자 |

두 학파는 적대적인 관계가 아니다. 우리는 상황에 맞는 최적의 도구를 선택하는 실용적인 전문가가 되어야 한다.

  * **상태 기반 검증이 유용할 때:** 시스템의 핵심 비즈니스 로직, 즉 순수한 계산이나 알고리즘을 다루는 **도메인 모델**을 테스트할 때 매우 효과적이다. 최종 결과가 중요하지, 그 결과를 얻기 위해 내부적으로 어떤 private 메소드를 호출하는지는 중요하지 않기 때문이다. (3장에서 집중적으로 사용)
  * **행위 기반 검증이 유용할 때:** 자신은 별다른 로직 없이 여러 협력 객체를 조율(Orchestration)하는 역할만 하는 객체를 테스트할 때 빛을 발한다. 예를 들어, 외부 API 호출 결과를 DB에 저장하는 서비스 객체의 경우, "외부 API 클라이언트를 호출했는가?"와 "Repository의 save 메소드를 호출했는가?"를 검증하는 것이 핵심이다. (4장, 5장에서 집중적으로 사용)

결론적으로, 우리는 두 가지 접근법을 모두 이해하고, 테스트하려는 대상의 **책임**이 무엇인지에 따라 적절한 검증 방식을 선택해야 한다. 그것이 바로 TDD를 통해 견고하고 유연한 설계를 만들어가는 장인의 길이다.