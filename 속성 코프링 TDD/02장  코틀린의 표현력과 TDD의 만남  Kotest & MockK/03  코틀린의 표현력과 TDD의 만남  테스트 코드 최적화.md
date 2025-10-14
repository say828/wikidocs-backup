## 03\. 코틀린의 특성(Extension Function, Data Class)을 활용한 테스트 코드 최적화

우리는 Kotest의 서술적 문법과 MockK의 강력한 제어 능력을 손에 넣었다. 하지만 우리의 무기고에는 아직 코틀린이라는 언어 자체가 제공하는 강력한 무기들이 남아있다. TDD 전문가의 코드는 단순히 동작하는 것을 넘어, 언어의 특성을 깊이 이해하고 그것을 적극적으로 활용하여 가독성과 유지보수성을 극한으로 끌어올린다.

이 절에서는 코틀린의 대표적인 두 가지 특성, \*\*확장 함수(Extension Function)\*\*와 \*\*데이터 클래스(Data Class)\*\*를 활용하여 우리의 테스트 코드를 어떻게 더 깔끔하고, 더 의미 있고, 더 효율적으로 만들 수 있는지 알아본다.

### **나만의 단언문 만들기: 확장 함수를 이용한 Custom Matcher**

Kotest는 `shouldBe`, `shouldHaveSize` 등 풍부한 Matcher를 기본으로 제공한다. 하지만 우리 도메인에 특화된, 반복적으로 사용되는 검증 로직이 있다면 어떨까? 예를 들어, `User` 객체의 `level`이 특정 등급 이상인지를 검증하는 로직이 여러 테스트에 걸쳐 사용된다고 가정해보자.

**나쁜 예: 일반적인 단언문을 반복 사용**

```kotlin
// 테스트 1
assertTrue(user.level >= Level.VIP)

// 테스트 2
assertTrue(anotherUser.level >= Level.VIP)

// 테스트 3
assertTrue(premiumUser.level >= Level.VIP)
```

이 코드는 동작하지만, `user.level >= Level.VIP`라는 표현은 "사용자가 VIP 등급 이상이어야 한다"는 비즈니스 요구사항을 직관적으로 드러내지 못한다. 우리는 코틀린의 확장 함수를 사용해 이 검증 로직을 마치 Kotest의 기본 Matcher처럼 만들 수 있다.

**좋은 예: 확장 함수로 만든 나만의 Custom Matcher**

```kotlin
// test/kotlin/com/tddmaster/utils/CustomMatchers.kt
// 테스트 코드 전용 유틸리티 파일에 확장 함수를 정의한다.
infix fun User.shouldBeVipOrHigher(expectedLevel: Level) {
    if (this.level < expectedLevel) {
        throw AssertionError("Expected user level to be $expectedLevel or higher, but was ${this.level}")
    }
}

// ... 테스트 코드 ...
import com.tddmaster.utils.shouldBeVipOrHigher

@Test
fun `VIP 사용자는 프리미엄 혜택을 받아야 한다`() {
    val vipUser = User(level = Level.VIP)
    val vvipUser = User(level = Level.VVIP)

    // 훨씬 더 읽기 쉽고, 도메인 용어로 요구사항을 서술한다.
    vipUser shouldBeVipOrHigher Level.VIP
    vvipUser shouldBeVipOrHigher Level.VIP
}
```

이제 우리의 테스트 코드는 기술적인 단언(`assertTrue`)이 아닌, 비즈니스 도메인의 언어로 요구사항을 서술하게 되었다. 이처럼 확장 함수를 활용하면 테스트 코드의 가독성을 비약적으로 향상시키고, 복잡한 검증 로직을 재사용 가능한 형태로 캡슐화할 수 있다.

### **데이터 준비와 결과 비교의 혁신: 데이터 클래스**

테스트 코드 작성 시간의 상당 부분은 테스트 데이터를 준비하고, 테스트 실행 후의 결과 객체와 기대 값을 비교하는 데 소요된다. 코틀린의 \*\*데이터 클래스(Data Class)\*\*는 이 두 가지 작업을 극적으로 단순화시켜준다.

1.  **구조적 동등성 비교 (`equals()`):** 데이터 클래스는 주 생성자에 선언된 프로퍼티들을 기반으로 `equals()` 메소드를 자동으로 생성해준다. 덕분에 복잡한 DTO(Data Transfer Object)나 값 객체(Value Object)를 비교할 때, 프로퍼티 하나하나를 비교할 필요 없이 객체 전체를 한 번에 비교할 수 있다.

2.  **불변 객체와 `copy()`:** 테스트에서는 종종 기본이 되는 데이터에서 특정 필드 값만 살짝 바꾼 여러 버전의 데이터가 필요하다. 데이터 클래스의 `copy()` 메소드는 불변성(immutability)을 유지하면서도 이러한 데이터 변형을 매우 쉽게 만들어준다.

**예시: 사용자 생성 API 테스트**

```kotlin
// 프로덕션 코드 (Data Class)
data class UserCreationRequest(val name: String, val email: String, val age: Int)
data class UserResponse(val id: Long, val name: String, val email: String)

// ... 테스트 코드 ...
@Test
fun `사용자 생성 요청이 성공하면, 생성된 사용자 정보가 반환된다`() {
    // given
    val request = UserCreationRequest(name = "TDD Master", email = "test@tdd.com", age = 30)

    // when
    val response = userController.createUser(request)

    // then
    val expectedResponse = UserResponse(id = 1L, name = "TDD Master", email = "test@tdd.com")
    
    // 데이터 클래스 덕분에 객체 전체를 한 번에 비교할 수 있다.
    response shouldBe expectedResponse
}

@Test
fun `이름만 다른 사용자를 연속으로 생성한다`() {
    val baseRequest = UserCreationRequest(name = "Base User", email = "base@tdd.com", age = 25)

    // copy()를 사용하여 필요한 프로퍼티만 변경한 새로운 객체를 쉽게 생성한다.
    val request1 = baseRequest.copy(name = "User A")
    val request2 = baseRequest.copy(name = "User B")

    userController.createUser(request1)
    userController.createUser(request2)

    // ... 검증 로직 ...
}
```

만약 `UserResponse`가 일반 클래스였다면, `response.id shouldBe expectedResponse.id`, `response.name shouldBe expectedResponse.name` 과 같이 모든 필드를 개별적으로 비교해야 했을 것이다. 데이터 클래스는 이러한 불필요한 상용구 코드를 제거하여 테스트를 간결하고 핵심에 집중하게 만든다.

-----

이것으로 코틀린 생태계의 강력한 테스트 도구들을 마스터하기 위한 여정을 마친다. 우리는 Kotest를 통해 테스트의 구조를 잡고, MockK로 의존성을 제어하며, 코틀린 언어의 특성을 활용해 코드를 최적화하는 방법까지 배웠다. 이제 우리는 어떤 종류의 코드라도 테스트할 수 있는 완벽한 무기를 갖추었다.

다음 장부터는 이 무기들을 들고 실제 전투에 나설 것이다. 시스템의 가장 핵심적인 심장부, **도메인 계층**을 TDD로 정복하는 여정을 시작해보자.