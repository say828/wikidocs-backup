### Enum 클래스와 sealed 클래스: 제한된 클래스 계층 구조

프로그램을 설계할 때, 특정 타입이 가질 수 있는 값의 종류를 한정된 집합으로 제한하고 싶을 때가 있습니다. 예를 들어 '요일'은 월, 화, 수, 목, 금, 토, 일 7가지 값만 가져야 하고, '네트워크 요청 상태'는 로딩중, 성공, 실패 세 가지 상태 중 하나여야 합니다.

이처럼 **'가능한 모든 경우의 수가 정해져 있는'** 상황을 모델링하기 위해 코틀린은 **Enum 클래스**와 **Sealed 클래스**라는 두 가지 강력한 도구를 제공합니다.

-----

#### Enum 클래스 (Enum Classes): 정해진 값들의 집합

\*\*Enum 클래스 (열거형 클래스)\*\*는 서로 연관된 **상수(constant)들의 고정된 집합**을 정의할 때 사용합니다. `String`이나 `Int`와 같은 원시 타입을 사용하는 것에 비해, 오타로 인한 버그를 방지하고 코드의 의미를 명확하게 만들어주는 타입 안전성(Type-safety)을 보장합니다.

```kotlin
// 요일을 나타내는 Enum 클래스
enum class DayOfWeek {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}
```

##### `when`과의 환상적인 조합

Enum 클래스는 `when` 표현식과 함께 사용될 때 그 진가를 발휘합니다. `when`의 대상이 Enum 클래스 타입일 경우, 컴파일러는 `when`이 **모든 가능한 Enum 상수를 처리했는지 검사**할 수 있습니다. 만약 모든 경우를 다루었다면 `else` 분기를 생략해도 됩니다. 이는 향후 새로운 Enum 상수가 추가되었을 때, 관련 `when`문이 컴파일 오류를 발생시켜 개발자가 로직 수정을 잊지 않도록 강제하는 매우 강력한 안전장치입니다.

```kotlin
fun getDayMessage(day: DayOfWeek): String {
    return when (day) {
        DayOfWeek.MONDAY -> "한 주의 시작! 힘내세요."
        DayOfWeek.FRIDAY -> "드디어 금요일! 주말이 다가옵니다."
        DayOfWeek.SATURDAY, DayOfWeek.SUNDAY -> "즐거운 주말!"
        else -> "평범한 하루." // MONDAY, FRIDAY, SATURDAY, SUNDAY를 제외한 나머지 요일
    }
}
```

##### 프로퍼티와 메서드를 가진 Enum

코틀린의 Enum은 단순한 상수의 목록을 넘어, 일반 클래스처럼 프로퍼티와 메서드를 가질 수 있습니다.

```kotlin
enum class Color(val hexCode: String) {
    RED("#FF0000"),
    GREEN("#00FF00"),
    BLUE("#0000FF"); // 프로퍼티나 메서드가 있을 경우, 상수 목록 끝에 세미콜론(;) 필수

    fun isPrimaryColor(): Boolean {
        return this == RED || this == GREEN || this == BLUE
    }
}
```

-----

#### Sealed 클래스 (Sealed Classes): 진화된 Enum

Enum 클래스가 '정해진 값들의 집합'이라면, \*\*Sealed 클래스(봉인된 클래스)\*\*는 \*\*'정해진 하위 클래스들의 집합'\*\*을 정의합니다. `sealed` 키워드가 붙은 클래스는 자신을 상속받는 모든 자식 클래스를 컴파일 시점에 미리 알게 됩니다.

**규칙:** Sealed 클래스의 모든 직접적인 자식 클래스는 반드시 **같은 파일 안에** 선언되어야 합니다.

##### Enum을 넘어서: 각 상태가 다른 데이터를 가질 때

Sealed 클래스는 Enum과 비슷하지만 훨씬 더 유연합니다. Enum의 모든 상수는 같은 타입이지만, Sealed 클래스의 각 자식 클래스는 \*\*서로 다른 프로퍼티를 가진 독립적인 클래스(일반 클래스, data 클래스, object 등)\*\*가 될 수 있습니다. 이는 각 상태가 서로 다른 데이터를 가져야 하는 복잡한 상황을 모델링하는 데 완벽합니다.

네트워크 요청의 결과를 예로 들어 봅시다.

  * **Loading**: 상태 자체만 중요하고 추가 데이터는 필요 없습니다.
  * **Success**: 성공했으며, `성공 데이터`를 함께 가지고 있어야 합니다.
  * **Error**: 실패했으며, `에러 메시지`를 함께 가지고 있어야 합니다.

<!-- end list -->

```kotlin
// Result.kt 파일 내부
sealed class Result {
    object Loading : Result() // 데이터가 없는 상태는 object로 간결하게 표현
    data class Success(val data: String) : Result() // 성공 상태는 데이터를 가짐
    data class Error(val message: String) : Result() // 에러 상태는 메시지를 가짐
}
```

##### `when`과의 완벽한 시너지

Sealed 클래스 역시 `when` 표현식과 함께 사용될 때, 컴파일러가 모든 가능한 자식 클래스 타입을 처리했는지 검사해 줍니다. 따라서 `else` 분기 없이도 타입 안전성을 완벽하게 보장받을 수 있습니다.

```kotlin
fun handleResult(result: Result) {
    when (result) {
        is Result.Loading -> {
            println("데이터를 로딩하고 있습니다...")
        }
        is Result.Success -> {
            // Success 타입으로 스마트 캐스팅되어 data 프로퍼티에 안전하게 접근
            println("성공! 데이터: ${result.data}")
        }
        is Result.Error -> {
            // Error 타입으로 스마트 캐스팅되어 message 프로퍼티에 안전하게 접근
            println("실패! 메시지: ${result.message}")
        }
    } // 모든 자식 클래스를 처리했으므로 else가 필요 없습니다.
}
```

**결론적으로,**

  * **Enum 클래스**는 상태들이 추가적인 데이터를 갖지 않거나, 모두 동일한 형태의 데이터만 가질 때 사용합니다. (예: 요일, 방향)
  * **Sealed 클래스**는 각 상태가 서로 다른 종류의 데이터를 가져야 하는 더 복잡한 계층 구조를 표현할 때 사용합니다. (예: API 응답 결과, 화면 상태)

이 두 가지 도구는 `when`과 결합하여 코드의 안정성과 표현력을 극적으로 향상시키는 현대 코틀린 애플리케이션 아키텍처의 핵심 요소입니다.