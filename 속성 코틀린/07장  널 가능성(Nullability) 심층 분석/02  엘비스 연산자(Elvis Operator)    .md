### 엘비스 연산자(Elvis Operator): ?:

안전한 호출 연산자(`?.`)는 `null`을 만났을 때 `null`을 반환하여 NPE를 우아하게 피합니다. 하지만 `null` 대신 \*\*기본값(default value)\*\*을 사용하고 싶을 때는 어떻게 해야 할까요?

```kotlin
val name: String? = null
var displayName: String

if (name != null) {
    displayName = name
} else {
    displayName = "Guest"
}
```

이런 `if-else` 구문은 꽤나 번거롭습니다. 코틀린은 이 패턴을 단 하나의 연산자로 압축할 수 있는 재치있는 방법을 제공합니다. 바로 \*\*엘비스 연산자(`?:`)\*\*입니다.

이 연산자는 `?:` 모양이 로큰롤의 황제, 엘비스 프레슬리의 옆모습과 헤어스타일을 닮았다고 해서 붙여진 재미있는 이름입니다. 🎸

**`?:`의 동작 방식**

> "왼쪽의 표현식이 `null`이 아니면, 그 값을 그대로 사용하라. 만약 `null`이면, 오른쪽의 표현식을 값으로 사용하라."

-----

#### 기본 사용법

엘비스 연산자를 사용하면 위의 `if-else` 구문을 단 한 줄로 바꿀 수 있습니다.

```kotlin
val name: String? = null
val displayName: String = name ?: "Guest" // name이 null이므로 "Guest"를 사용
println("Hello, $displayName") // 출력: Hello, Guest

val realName: String? = "Alice"
val displayName2: String = realName ?: "Guest" // realName이 null이 아니므로 "Alice"를 사용
println("Hello, $displayName2") // 출력: Hello, Alice
```

엘비스 연산자 덕분에 `displayName` 변수의 타입이 `String?`가 아닌 `String`이 될 수 있습니다. 어떤 경우에도 `null`이 아닌 값이 보장되기 때문입니다.

-----

#### 안전한 호출 연산자(`?.`)와의 조합

엘비스 연산자의 진정한 위력은 안전한 호출 연산자(`?.`)와 함께 사용될 때 나타납니다. `?.` 체이닝의 결과가 `null`일 경우, 우아하게 기본값을 지정할 수 있습니다.

```kotlin
// Student -> Department -> Professor -> Name 구조를 다시 사용
val studentWithNoProfessor = Student(Department(null))

// 지도교수의 이름을 가져오되, 없으면 "담당 교수 미정"을 반환
val professorName = studentWithNoProfessor.department?.headProfessor?.name ?: "담당 교수 미정"

println("담당 교수: $professorName") // 출력: 담당 교수: 담당 교수 미정
```

`student.department?.headProfessor?.name` 부분이 `null`을 반환하자, 엘비스 연산자가 즉시 동작하여 "담당 교수 미정"이라는 기본값을 제공했습니다. 이 패턴은 복잡한 객체 그래프를 탐색할 때 매우 유용합니다.

-----

#### `return` 또는 `throw`와 함께 사용하기

엘비스 연산자의 오른쪽에는 단순한 값뿐만 아니라, `return`이나 `throw` 같은 구문도 올 수 있습니다. 이는 함수의 초반부에서 파라미터나 상태를 검사하는 '전제 조건' 코드를 작성할 때 매우 깔끔한 코드를 만들어 줍니다.

##### `return`과 함께 사용

```kotlin
fun printShippingLabel(customer: Customer?) {
    // customer가 null이면 함수를 즉시 종료(return)
    val address = customer?.address ?: return

    // 이 줄부터 컴파일러는 address가 null이 아님을 알고 있습니다.
    println(address.street)
    println(address.city)
}
```

##### `throw`와 함께 사용

```kotlin
fun processPayment(amount: Double?) {
    // amount가 null이면 예외를 발생(throw)
    val validAmount = amount ?: throw IllegalArgumentException("결제 금액은 null일 수 없습니다.")

    // 이 줄부터 validAmount는 Double 타입으로 안전하게 사용 가능
    println("결제를 진행합니다: $validAmount 원")
}
```

이처럼 엘비스 연산자는 `null`일 경우를 대비한 기본값을 제공하는 것을 넘어, 프로그램의 흐름을 제어하는 강력하고 간결한 도구입니다. `?.`와 `?:`를 조합하는 것은 널 안정성을 다루는 가장 코틀린스러운 방법 중 하나입니다.