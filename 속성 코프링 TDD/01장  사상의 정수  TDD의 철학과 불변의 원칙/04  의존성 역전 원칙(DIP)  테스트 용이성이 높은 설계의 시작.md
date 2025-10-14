## 04\. 의존성 역전 원칙(DIP): 테스트 용이성이 높은 설계의 시작

지금까지 TDD의 철학과 원칙들을 살펴보았다. 하지만 이 모든 것을 현실의 코드에 적용하려고 할 때, 많은 개발자들이 첫 번째 장벽에 부딪힌다. 바로 \*\*"이 코드는 테스트하기가 너무 어렵습니다"\*\*라는 문제다. 테스트하기 어려운 코드는 우연히 만들어지는 것이 아니다. 그것은 잘못된 설계, 특히 객체 간의 의존성 관리가 실패한 결과물이다.

테스트 용이성이 높은, 즉 TDD를 적용하기 좋은 코드를 만드는 가장 근본적인 설계 원칙은 바로 객체 지향 설계의 5원칙 **SOLID** 중 마지막, \*\*의존성 역전 원칙(DIP, Dependency Inversion Principle)\*\*이다. TDD와 DIP는 서로를 이끌어주는 완벽한 파트너 관계다.

### **DIP 이전의 세계: 강한 결합의 비극**

DIP가 적용되지 않은 코드는 어떤 모습일까? `OrderService`가 주문을 처리하기 위해 `MySqlProductRepository`를 사용한다고 가정해보자.

**나쁜 예: DIP를 위반하는 코드**

```kotlin
// 구체적인 클래스 MySqlProductRepository
class MySqlProductRepository {
    fun findById(id: String): Product {
        // 실제 MySQL에 연결해서 데이터를 가져오는 복잡한 로직
        println("Connecting to MySQL to find product $id...")
        // ...
        return Product(id, "TDD 티셔츠", 10)
    }
}

// 상위 수준 모듈인 OrderService
class OrderService {
    // 상위 수준 모듈이 하위 수준의 '구체적인' 클래스에 직접 의존한다.
    private val productRepository: MySqlProductRepository = MySqlProductRepository()

    fun placeOrder(productId: String, quantity: Int) {
        val product = productRepository.findById(productId)
        // ... 주문 처리 로직
    }
}
```

이 `OrderService`를 단위 테스트하는 것은 거의 불가능하다. 테스트를 실행할 때마다 `MySqlProductRepository`가 생성되어 실제 MySQL 데이터베이스 연결을 시도할 것이기 때문이다. 이는 **FIRST** 원칙의 **Fast**, **Independent**, **Repeatable** 모두를 위반한다. 상위 정책을 담은 `OrderService`가 하위 구현의 세부사항인 `MySqlProductRepository`에 강하게 묶여(결합되어) 있기 때문이다.

-----

### **DIP의 구원: 추상화에 의존하라**

의존성 역전 원칙은 이 문제를 해결하기 위해 두 가지를 제안한다.

1.  상위 수준 모듈은 하위 수준 모듈에 의존해서는 안 된다. 둘 모두 \*\*추상화(Abstraction)\*\*에 의존해야 한다.
2.  추상화는 세부 사항에 의존해서는 안 된다. 세부 사항(구체적인 구현체)이 추상화에 의존해야 한다.

여기서 '역전'이라는 단어가 의미하는 것은, 전통적인 방식처럼 상위 모듈이 하위 모듈을 선택하고 제어하는 의존성의 방향을 뒤집는다는 것이다. 제어의 주체를 상위 모듈에서 외부(제3자)로 옮긴다.

**좋은 예: DIP를 적용하여 리팩토링한 코드**

1.  **추상화(인터페이스)를 정의한다.**

<!-- end list -->

```kotlin
interface ProductRepository {
    fun findById(id: String): Product
}
```

2.  **세부 사항(구체 클래스)이 추상화에 의존(구현)하도록 한다.**

<!-- end list -->

```kotlin
class MySqlProductRepository : ProductRepository {
    override fun findById(id: String): Product {
        // ... 실제 MySQL 로직 ...
        return Product(id, "TDD 티셔츠", 10)
    }
}
```

3.  **상위 수준 모듈이 추상화에 의존하도록 한다.**

<!-- end list -->

```kotlin
class OrderService(
    // 이제 구체 클래스가 아닌 인터페이스에 의존한다.
    // 의존성은 외부에서 '주입(Inject)' 받는다.
    private val productRepository: ProductRepository 
) {
    fun placeOrder(productId: String, quantity: Int) {
        val product = productRepository.findById(productId)
        // ... 주문 처리 로직
    }
}
```

### **DIP가 가져온 선물: 완벽한 테스트 용이성**

이제 `OrderService`를 테스트하는 것은 놀랍도록 간단해진다. 우리는 더 이상 실제 데이터베이스를 필요로 하지 않는다. 대신, 테스트만을 위한 가짜 구현체(Fake Object)를 만들어 `OrderService`에 주입하면 그만이다.

```kotlin
class FakeInMemoryProductRepository : ProductRepository {
    private val products = mutableMapOf<String, Product>()
    
    fun save(product: Product) {
        products[product.id] = product
    }

    override fun findById(id: String): Product {
        return products[id] ?: throw Exception("Product not found")
    }
}

@Test
fun `OrderService는 ProductRepository를 통해 상품을 조회한다`() {
    // given: 테스트용 가짜 Repository를 준비한다.
    val fakeRepository = FakeInMemoryProductRepository()
    fakeRepository.save(Product("P001", "TDD 티셔츠", 10))
    
    // when: OrderService에 가짜 Repository를 '주입'하여 생성하고 테스트한다.
    val orderService = OrderService(fakeRepository)
    orderService.placeOrder("P001", 2)
    
    // then: 테스트는 빠르고, 독립적이며, 반복 가능하다.
    // (검증 로직 추가)
}
```

**TDD는 우리에게 의존성을 격리할 것을 끊임없이 요구한다.** 이 요구에 응답하는 가장 확실하고 올바른 방법이 바로 의존성 역전 원칙을 적용하는 것이다. 결국, TDD는 그 자체로 훌륭한 객체 지향 설계 원칙을 따르도록 우리를 이끄는 훈련 과정인 셈이다.

이것으로 TDD의 철학과 불변의 원칙에 대한 탐험을 마친다. 우리는 TDD의 리듬과 전략, 품질 기준, 그리고 TDD를 가능하게 하는 설계 원칙까지 단단한 이론적 기반을 갖추었다. 이제 이 모든 것을 현실의 코드로 구현할 시간이다. 다음 장에서는 우리의 손과 발이 되어줄 강력한 테스트 도구, Kotest와 MockK의 세계로 떠나보자.