### 코틀린에서의 단위 테스트: JUnit, Mockito, Kotest

코틀린은 JVM 위에서 동작하므로, 자바의 거대하고 성숙한 테스트 생태계를 100% 활용할 수 있습니다. 동시에, 코틀린은 언어의 특성을 십분 살린 더 현대적이고 강력한 테스트 도구들을 제공합니다.

#### 1\. JUnit 5: 테스트 실행의 표준

**JUnit**은 자바 생태계의 표준 테스트 프레임워크이자, 테스트를 실행하는 '런처(Launcher)' 역할을 합니다. 코틀린은 JUnit 5와 완벽하게 호환되며, 코틀린의 언어적 특징을 더해 테스트 코드를 훨씬 더 깔끔하게 작성할 수 있습니다.

  * **백틱(Backtick)을 이용한 테스트 함수명:** 코틀린에서는 함수 이름을 백틱( `` ` `` )으로 감싸면, 공백과 특수문자를 포함한 자연어 문장을 함수 이름으로 사용할 수 있습니다. 이는 테스트의 의도를 매우 명확하게 설명해 줍니다.
  * **간결한 구문:** `public` 키워드나 불필요한 클래스 상속 없이 깔끔한 테스트 작성이 가능합니다.

<!-- end list -->

```kotlin
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals

class CalculatorTest {

    private val calculator = Calculator()

    @Test
    fun `1과 2를 더하면 3을 반환해야 한다`() {
        // given (준비)
        val a = 1
        val b = 2

        // when (실행)
        val result = calculator.add(a, b)

        // then (검증)
        assertEquals(3, result, "1 + 2는 3이어야 합니다.")
    }
}
```

#### 2\. Mockito와 `mockito-kotlin`: 의존성 격리하기

\*\*단위 테스트(Unit Test)\*\*의 핵심은 테스트 대상을 주변 의존성(데이터베이스, 네트워크 등)으로부터 '격리'하는 것입니다. 이를 위해 가짜 객체, 즉 \*\*목(Mock)\*\*을 만들어 사용하며, **Mockito**는 자바 생태계에서 가장 널리 사용되는 모킹 라이브러리입니다.

하지만 코틀린의 클래스와 메서드는 기본적으로 `final` (상속/오버라이드 불가)이기 때문에, 런타임에 프록시 객체를 생성하는 Mockito의 기본 방식과 충돌합니다.

이 문제를 해결하는 방법은 두 가지입니다.

1.  **`mockito-inline`:** Mockito 설정에 이 기능을 추가하면, `final` 클래스와 메서드도 마법처럼 모킹할 수 있게 됩니다.
2.  **`mockito-kotlin`:** 코틀린에서 Mockito를 훨씬 더 관용적으로 사용할 수 있게 돕는 래퍼(wrapper) 라이브러리입니다. `mock()`, `whenever()` 등의 함수를 제공하여 코틀린 문법에 잘 어울리는 코드를 작성하게 해줍니다.

<!-- end list -->

```kotlin
import org.mockito.kotlin.mock
import org.mockito.kotlin.whenever
import org.mockito.kotlin.verify

// UserService가 UserRepository에 의존한다고 가정
class UserServiceTest {

    private val mockRepository: UserRepository = mock() // mockito-kotlin 헬퍼 함수
    private val userService = UserService(mockRepository)

    @Test
    fun `사용자 ID로 사용자 이름을 가져와야 한다`() {
        // given: 1번 ID로 조회하면 "Alice"를 반환하도록 설정
        whenever(mockRepository.findUserById(1)).thenReturn(User(1, "Alice"))

        // when
        val userName = userService.getUserName(1)

        // then
        assertEquals("Alice", userName)
        verify(mockRepository).findUserById(1) // findUserById(1)이 1번 호출되었는지 검증
    }
}
```

#### 3\. Kotest: 코틀린 네이티브 테스트 프레임워크

JUnit과 Mockito가 '자바 생태계를 코틀린에서 활용하는 법'이라면, **Kotest**는 '코틀린을 위해 코틀린으로 만들어진' 가장 대표적인 차세대 테스트 프레임워크입니다.

Kotest는 단순한 테스트 러너를 넘어, 강력한 DSL과 풍부한 기능을 통합 제공합니다.

  * **표현력 있는 DSL:** `StringSpec`, `BehaviorSpec` 등 다양한 테스트 스타일을 제공하여, 테스트 코드를 자연어 명세서처럼 작성할 수 있습니다.
  * **강력한 단언(Assertions) 라이브러리:** `assertEquals`를 넘어, `result shouldBe 3`, `list shouldHaveSize 5`, `string shouldStartWith "hello"` 처럼 읽기 쉬운 수많은 검증 함수를 내장하고 있습니다.
  * **데이터 주도 테스트:** 여러 입력값과 기대값을 테이블 형태로 제공하여 테스트를 자동 반복 실행하는 기능이 매우 강력합니다.

<!-- end list -->

```kotlin
import io.kotest.core.spec.style.BehaviorSpec
import io.kotest.matchers.shouldBe

class CalculatorKotest : BehaviorSpec({
    
    val calculator = Calculator()

    Given("계산기가 주어졌을 때") {
        When("1과 2를 더하면") {
            val result = calculator.add(1, 2)
            Then("결과는 3이어야 한다") {
                result shouldBe 3 // Kotest의 직관적인 단언문
            }
        }
        
        When("5에서 3을 빼면") {
            val result = calculator.subtract(5, 3)
            Then("결과는 2여야 한다") {
                result shouldBe 2
            }
        }
    }
})
```

코틀린 프로젝트를 시작한다면, 자바의 유산을 활용하는 JUnit + Mockito 조합도 훌륭하지만, 코틀린의 언어적 특성을 극대화하는 Kotest를 도입하는 것은 코드 품질과 테스트 작성 경험을 한 차원 높여주는 최고의 선택이 될 수 있습니다.