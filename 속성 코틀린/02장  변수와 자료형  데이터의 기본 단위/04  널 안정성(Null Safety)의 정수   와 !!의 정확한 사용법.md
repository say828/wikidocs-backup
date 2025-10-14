### 널 안정성(Null Safety)의 정수: ?와 \!\!의 정확한 사용법

프로그래밍의 역사상 가장 많은 버그를 유발한 주범을 꼽으라면 단연 `NullPointerException`(NPE)일 것입니다. `null` 참조를 처음 고안한 토니 호어(Tony Hoare)조차 이를 '억만 달러짜리 실수'라고 불렀을 정도입니다. `null`은 '값이 존재하지 않음'을 나타내는 유용한 개념이지만, 언제 어디서 터질지 모르는 시한폭탄과도 같았습니다. 개발자는 값이 `null`일 가능성을 항상 염두에 두고 방어 코드를 작성해야 했고, 이를 놓치면 프로그램은 사용자의 눈앞에서 무참히 멈춰 섰습니다.

코틀린은 이 고질적인 문제를 언어 설계 차원에서 해결하기 위해 칼을 빼 들었습니다. 바로 **널 안정성(Null Safety)** 이라는 매우 강력한 시스템입니다.

#### 패러다임의 전환: Null을 허용하지 않는 타입

코틀린의 접근 방식은 근본부터 다릅니다. **코틀린의 모든 타입은 기본적으로 `null` 값을 가질 수 없습니다(Non-nullable).**

```kotlin
var name: String = "Kotlin"
// name = null // 컴파일 오류! String 타입의 변수에는 null을 할당할 수 없습니다.
```

컴파일러는 `String` 타입의 변수 `name`에는 절대로 `null`이 들어갈 수 없음을 보장합니다. 따라서 우리는 `name` 변수를 사용할 때 `null`인지 아닌지 검사할 필요 없이, 항상 안전하게 `String` 객체라고 믿고 사용할 수 있습니다.

#### 물음표(`?`): Null 가능성에 대한 허가

그렇다면 `null` 값을 정말 담아야 할 때는 어떻게 해야 할까요? 이때 개발자는 자신의 의도를 컴파일러에게 명확히 밝혀야 합니다. 바로 타입 이름 뒤에 \*\*물음표(`?`)\*\*를 붙이는 것입니다.

```kotlin
var nullableName: String? = "Kotlin" // 'String?'은 null이 가능한 String 타입입니다.
println(nullableName) // 출력: Kotlin

nullableName = null // OK.
println(nullableName) // 출력: null
```

`String?`는 '`String`일 수도 있고, `null`일 수도 있는 타입'을 의미합니다. 이렇게 `?`를 붙여 Nullable 타입을 선언하는 순간, 개발자는 컴파일러와 다음과 같은 계약을 맺게 됩니다.
**"나는 이 변수가 `null`일 수 있음을 인지하고 있으며, 앞으로 이 변수를 사용할 때 `null`일 경우를 반드시 책임지고 처리하겠습니다."**

-----

#### Nullable 타입 안전하게 사용하기

이제 컴파일러는 `String?` 타입의 변수를 사용할 때마다 우리가 계약을 잘 지키는지 감시하기 시작합니다.

```kotlin
var name: String? = null
// println(name.length) // 컴파일 오류! 
```

위 코드는 컴파일되지 않습니다. 만약 `name`이 `null`인 상태에서 `.length`를 호출하면 `NullPointerException`이 발생할 것이 뻔하기 때문입니다. 컴파일러가 이 위험한 코드를 사전에 차단해 주는 것입니다. 그렇다면 어떻게 이 변수를 안전하게 사용할 수 있을까요?

##### 1\. 안전한 호출 연산자 (Safe Call Operator): `?.`

가장 일반적이고 권장되는 방법입니다. `?.` 연산자는 `null` 검사와 함수 호출을 한 번에 처리합니다.

**"만약 객체가 `null`이 아니면, 뒤따르는 멤버에 접근하고, `null`이라면 그냥 `null`을 결과로 내놓으라."**

```kotlin
var name: String? = "Kotlin"
println(name?.length) // 출력: 6

var otherName: String? = null
println(otherName?.length) // 출력: null (NPE가 발생하지 않고, null이 출력됩니다.)
```

`?.` 하나만으로 `if (name != null) name.length else null`과 같은 코드가 간결하게 표현됩니다.

##### 2\. 비-널 단정 연산자 (Not-Null Assertion Operator): `!!`

때로는 개발자가 특정 변수가 이 코드 라인에서는 **절대로 `null`일 리 없다**고 100% 확신할 수 있는 경우가 있습니다. 이럴 때 사용하는 것이 `!!` 연산자입니다.

**"컴파일러, 너의 걱정은 알겠지만 나를 믿어라\! 이 값은 절대 `null`이 아니니, 그냥 Non-null 타입으로 간주하고 실행해\!"**

`!!`는 Nullable 타입을 강제로 Non-null 타입으로 바꿔버립니다.

```kotlin
fun getFixedName(): String? = "Kotlin"

val name: String? = getFixedName()
val length: Int = name!!.length // 개발자가 name이 절대 null이 아니라고 보증합니다.
println(length) // 출력: 6
```

**하지만 매우 주의해야 합니다\!** `!!`는 컴파일러의 안전장치를 개발자가 스스로 해제하는 행위입니다. 만약 개발자의 보증이 틀려서 해당 값이 `null`이었다면, `!!` 연산자는 어김없이 `NullPointerException`을 발생시킵니다.

> **`!!` 연산자는 언제 사용해야 할까요?** \> **정답은 '가급적 절대 사용하지 않는다'입니다.** `!!`를 사용하고 싶다는 생각이 든다면, 그것은 코드 설계를 개선하여 `null` 가능성을 원천적으로 제거할 수 있다는 신호일 때가 많습니다. `!!`는 최후의 수단이며, 코드에 `!!`가 보인다면 그 이유를 반드시 주석으로 남기는 것이 좋습니다.

코틀린의 널 안정성은 `null`을 없애는 것이 아니라, `null`을 시스템의 통제하에 두어 다루는 것입니다. `?`를 통해 `null`의 위험성을 명시하고, `?.`를 통해 안전하게 다루는 것이 코틀린이 제시하는 현명한 길입니다.