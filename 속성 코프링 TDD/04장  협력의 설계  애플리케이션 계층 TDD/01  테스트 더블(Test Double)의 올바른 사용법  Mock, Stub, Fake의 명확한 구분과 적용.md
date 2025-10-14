## 01\. 테스트 더블(Test Double)의 올바른 사용법: Mock, Stub, Fake의 명확한 구분과 적용

애플리케이션 계층의 테스트는 '협력'을 검증하는 것이라고 했다. 하지만 어떻게 실제 데이터베이스나 외부 API 없이 이 협력을 테스트할 수 있을까? 영화 촬영에서 위험한 장면을 실제 배우 대신 스턴트 배우가 연기하듯, 테스트에서도 실제 의존 객체(Dependency)를 대신하는 가짜 객체를 사용한다. 이 모든 가짜 객체를 통칭하여 \*\*테스트 더블(Test Double)\*\*이라고 부른다.

많은 개발자들이 'Mock'이라는 단어를 모든 가짜 객체를 지칭하는 용어로 혼용하지만, 이는 TDD의 의도를 흐리는 위험한 습관이다. 마틴 파울러는 테스트 더블을 목적에 따라 여러 종류로 구분했으며, TDD 전문가는 이들을 명확히 구분하고 올바른 상황에 올바른 더블을 사용할 줄 알아야 한다. 그중 가장 핵심적인 **Stub, Mock, Fake**에 대해 알아보자.

-----

### **Stub: 상태를 제어하는 배우**

  * **목적:** 테스트 중에 만들어진 호출에 대해 미리 준비된 답변을 제공하는 것이 주된 목적이다. 스텁은 테스트 대상 객체(SUT)가 정상적으로 동작하기 위해 필요한 상태나 값을 제공하는 역할을 한다.
  * **검증 방식:** \*\*상태 기반 검증(State-based Verification)\*\*과 함께 쓰인다. 테스트는 스텁이 어떻게 '사용'되었는지에는 관심이 없다. 오직 SUT가 스텁이 제공한 데이터를 바탕으로 올바른 결과를 반환했는지, 또는 올바른 상태 변화를 일으켰는지만을 검증한다.
  * **비유:** 스텁은 정해진 대사만 말하는 단역 배우와 같다. 주연 배우(SUT)가 "오늘 날씨는?"이라고 물으면, 스텁은 감독(테스트 코드)이 시킨 대로 "맑음"이라고 대답할 뿐이다. 우리는 주연 배우가 "맑음"이라는 대답을 듣고 우산을 챙기지 않는 '행동'의 결과를 볼 뿐, 단역 배우에게 몇 번 말을 걸었는지는 신경 쓰지 않는다.

**예시: `ProductService`가 Stub `Repository`로부터 가격 정보를 받아오는 테스트**

```kotlin
@Test
fun `상품 가격에 배송비를 더한 총액을 계산해야 한다`() {
    // given: ProductRepository를 Stub으로 만든다.
    val stubProductRepository = mockk<ProductRepository>()
    val productService = ProductService(stubProductRepository)
    val product = Product(id = "P001", price = Money(BigDecimal(10000)))
    
    // 'findById'가 호출되면 미리 준비된 'product' 객체를 반환하라고 지시한다.
    every { stubProductRepository.findById("P001") } returns product
    
    // when: 가격 계산 로직을 실행
    val totalPrice = productService.calculateTotalPrice("P001")
    
    // then: SUT의 '상태' 또는 '결과'를 검증한다. Stub이 어떻게 호출되었는지는 검증하지 않는다.
    totalPrice shouldBe Money(BigDecimal(12500)) // 배송비 2500원
}
```

-----

### **Mock: 행위를 검증하는 감독관**

  * **목적:** 객체가 받은 호출에 대해 스스로를 검증하는 것이 주된 목적이다. 즉, SUT가 협력 객체와 올바른 방식으로 '상호작용'했는지를 추적하고 기록한다.
  * **검증 방식:** \*\*행위 기반 검증(Behavior-based Verification)\*\*과 함께 쓰인다. 테스트의 마지막에 `verify` 구문을 통해 "이 메소드가 정확히 N번 호출되었는가?", "이 메소드가 올바른 인자와 함께 호출되었는가?"를 검증한다.
  * **비유:** 목은 운전면허 시험의 감독관과 같다. 감독관은 운전자(SUT)가 좌회전을 할 때, "좌측 방향 지시등을 켰는가?", "사이드 미러를 확인했는가?"와 같은 '행위' 자체를 하나하나 관찰하고 채점한다. 차가 실제로 좌회전을 성공했는지(결과)보다, 정해진 절차(상호작용)를 따랐는지가 더 중요하다.

**예시: `UserService`가 회원 가입 시 Mock `EmailClient`를 호출하는지 검증**

```kotlin
@Test
fun `회원 가입에 성공하면 환영 이메일을 보내야 한다`() {
    // given: EmailClient를 Mock으로 만든다.
    val mockEmailClient = mockk<EmailClient>(relaxed = true)
    val userService = UserService(mockEmailClient)
    
    // when: 회원 가입 로직 실행
    userService.register("test@example.com")
    
    // then: SUT의 '행위'를 검증한다. EmailClient의 send 메소드가 호출되었는지를 확인한다.
    verify(exactly = 1) {
        mockEmailClient.send(to = "test@example.com", subject = "환영합니다!")
    }
}
```

-----

### **Fake: 실제처럼 동작하는 대역**

  * **목적:** 실제 구현을 대체할 수 있는, 동작하는 코드를 가진 객체다. 하지만 프로덕션 환경에서 사용하기에는 부족한, 테스트에 적합한 가벼운 구현체다. 대표적인 예가 실제 데이터베이스 대신 메모리 상의 `HashMap`을 사용하는 '가짜' 리포지토리다.
  * **비유:** 페이크는 비행기 조종사 훈련에 사용되는 '플라이트 시뮬레이터'와 같다. 시뮬레이터는 실제 비행기는 아니지만, 이륙, 비행, 착륙 등 실제와 매우 유사하게 동작하는 '로직'을 가지고 있다.
  * **구현:** 보통 MockK와 같은 라이브러리로 만들기보다는, 테스트 소스셋에 직접 인터페이스를 구현하는 클래스를 작성한다.

**예시: `FakeInMemoryUserRepository`**

```kotlin
// test 소스셋에 직접 작성
class FakeInMemoryUserRepository : UserRepository {
    private val users = mutableMapOf<Long, User>()
    private var nextId = 1L

    override fun save(user: User): User {
        val savedUser = user.copy(id = nextId++)
        users[savedUser.id!!] = savedUser
        return savedUser
    }
    
    override fun findByEmail(email: String): User? {
        return users.values.find { it.email == email }
    }
}
```

| 종류 | 주요 목적 | 검증 스타일 | 구현 방식 |
| :--- | :--- | :--- | :--- |
| **Stub** | SUT에 필요한 값을 제공 | 상태 기반 | Mocking 라이브러리 (e.g., MockK) |
| **Mock** | SUT와의 상호작용(호출) 검증 | 행위 기반 | Mocking 라이브러리 (e.g., MockK) |
| **Fake** | 실제 구현의 가벼운 대체 | 상태/행위 모두 | 직접 클래스 작성 |

어떤 테스트 더블을 선택할지는 우리가 '무엇을 검증하고 싶은가'에 달려있다. 결과 값을 보고 싶다면 스텁을, 특정 행동을 했는지 확인하고 싶다면 목을, 여러 객체 간의 복잡한 상호작용을 시뮬레이션하고 싶다면 페이크를 사용해야 한다. 이들을 올바르게 구분하여 사용하는 것이 바로 견고하고 의미 있는 애플리케이션 계층 테스트의 첫걸음이다.