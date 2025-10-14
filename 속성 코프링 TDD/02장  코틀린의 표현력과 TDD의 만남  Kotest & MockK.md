# 02장: 코틀린의 표현력과 TDD의 만남: Kotest & MockK

1장에서 우리는 TDD를 지탱하는 단단한 철학적 기반을 다졌다. `Red-Green-Refactor`의 리듬, 테스트 피라미드라는 전략, 그리고 테스트 용이성을 높이는 DIP와 같은 설계 원칙까지, 우리는 이제 '왜' TDD를 해야 하는지에 대한 깊은 이해를 갖추게 되었다.

이제 그 철학을 현실의 코드로 빚어낼 시간이다. 이 장에서는 코틀린의 잠재력을 최대한 끌어내어 우리의 테스트 코드를 단순한 검증 스크립트에서 \*\*'살아있는 명세서'\*\*로 격상시켜 줄 두 가지 핵심 도구, **Kotest**와 **MockK**를 완벽하게 마스터할 것이다. 스프링 부트가 기본으로 제공하는 JUnit 5와 Mockito도 훌륭한 도구지만, 코틀린이라는 표현력 넘치는 언어를 사용하면서 굳이 과거의 방식에 머물러 있을 이유는 없다. Kotest와 MockK는 코틀린을 위해, 코틀린에 의해 만들어진 만큼, 우리가 1장에서 배운 모든 원칙들을 가장 코틀린답고 우아하며 효율적인 방식으로 구현할 수 있도록 도와줄 것이다.

-----

## 00\. JUnit 5를 넘어서: Kotest가 제공하는 서술적 테스트의 힘

스프링 부트 프로젝트를 시작하면 `spring-boot-starter-test` 의존성을 통해 JUnit 5가 자동으로 포함된다. JUnit 5는 JVM 생태계의 표준 테스트 프레임워크이며, 그 자체로도 매우 강력하고 성숙한 도구다. 하지만 코틀린 개발자의 관점에서 볼 때, 테스트를 작성하는 방식에 있어 한 가지 아쉬운 점이 있다. 바로 \*\*'서술적인 표현력'\*\*이다.

JUnit 5의 기본적인 테스트는 다음과 같은 형태를 띤다.

**일반적인 JUnit 5 테스트 코드**

```kotlin
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.assertEquals

class CalculatorTest {

    @Test
    fun `덧셈을 할 경우 두 숫자의 합을 반환해야 한다`() {
        // given
        val calculator = Calculator()
        
        // when
        val result = calculator.add(3, 5)
        
        // then
        assertEquals(8, result)
    }
}
```

위 코드는 기능적으로 아무런 문제가 없다. 하지만 백틱(` `` `)을 사용한 함수명은 어딘가 부자연스럽고, 테스트의 구조가 평면적이어서 복잡한 시나리오를 표현하기에는 한계가 있다. 우리는 1장에서 "테스트는 곧 명세서"라고 배웠다. 그렇다면 우리의 테스트 코드도 명세서처럼, 혹은 잘 쓰인 한 편의 이야기처럼 읽혀야 하지 않을까?

이것이 바로 **Kotest**가 필요한 이유다. Kotest는 JUnit 5 플랫폼 위에서 동작하는 테스트 프레임워크로, 개발자가 다양한 스타일을 통해 테스트의 의도를 훨씬 명확하고 서술적으로 표현할 수 있도록 돕는다.

**Kotest의 `BehaviorSpec`을 사용한 동일한 테스트 코드**

```kotlin
import io.kotest.core.spec.style.BehaviorSpec
import io.kotest.matchers.shouldBe

class CalculatorKotest : BehaviorSpec({
    // given: 계산기가 주어졌을 때
    given("a calculator") {
        val calculator = Calculator()

        // when: 3과 5를 더하면
        `when`("we add 3 and 5") {
            val result = calculator.add(3, 5)

            // then: 결과는 8이 되어야 한다
            then("it should return 8") {
                result shouldBe 8
            }
        }
    }
})
```

### **무엇이 달라졌는가?**

1.  **명확한 구조 (Clear Structure):** `given-when-then` 블록은 BDD(행위 주도 개발)에서 유래한 구조로, 테스트의 **맥락(Context)**, **행위(Action)**, \*\*검증(Verification)\*\*을 명확하게 분리한다. 이 중첩 구조 덕분에 복잡한 시나리오도 논리적으로 그룹화하여 표현할 수 있다.
2.  **뛰어난 가독성 (Superior Readability):** 코드가 마치 자연어 문장처럼 읽힌다. "계산기가 주어졌을 때, 3과 5를 더하면, 결과는 8이 되어야 한다." 이는 테스트의 의도를 파악하는 데 드는 인지적 비용을 극적으로 낮춰준다.
3.  **풍부한 Matcher (Rich Matchers):** `assertEquals(8, result)` 대신 `result shouldBe 8`과 같은 Kotest의 확장 함수 기반 Matcher를 사용한다. 이는 코드를 더 자연스럽게 읽히도록 만들며, 이 외에도 컬렉션, 문자열, 예외 등을 위한 수많은 편리한 Matcher를 제공한다.

Kotest는 단순히 문법적 설탕(syntactic sugar)을 제공하는 것을 넘어, 우리의 사고방식을 바꾼다. '함수 실행 결과를 확인하는 코드'를 작성하는 것에서 \*\*'시스템의 특정 행위를 서술하는 명세'\*\*를 작성하는 것으로 말이다.

이처럼 서술적인 테스트 코드는 유지보수가 용이하며, 팀의 새로운 구성원도 테스트 코드만 읽고도 시스템의 요구사항을 쉽게 파악할 수 있게 돕는다. 우리는 이제 JUnit 5라는 단단한 기반 위에, Kotest라는 강력한 표현력을 더해 우리의 테스트를 한 차원 높은 수준으로 끌어올릴 준비를 마쳤다.