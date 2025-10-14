### 함수 타입(Function Types)의 이해

우리는 `val number: Int` 처럼 변수를 선언할 때, 그 변수가 어떤 종류의 데이터를 담을 것인지를 **타입**으로 명시합니다. 그렇다면 함수를 변수에 할당하거나 다른 함수에 전달할 때, 그 '함수'라는 데이터의 타입은 어떻게 표현할까요?

코틀린은 이를 위해 \*\*함수 타입(Function Type)\*\*이라는 특별한 문법을 제공합니다. 함수 타입은 함수의 **시그니처(Signature)**, 즉 어떤 타입의 파라미터를 받고 어떤 타입을 반환하는지를 나타내는 명세입니다.

-----

#### 함수 타입의 문법

함수 타입의 기본 구조는 매우 직관적입니다.

`(파라미터_타입_목록) -> 반환_타입`

  * **`( ... )`**: 괄호 안에는 함수가 받을 파라미터들의 타입을 쉼표(`,`)로 구분하여 나열합니다. 파라미터가 없으면 빈 괄호`()`를 사용합니다.
  * **`->`**: 화살표는 파라미터와 반환 타입을 구분합니다.
  * **`반환_타입`**: 함수가 최종적으로 반환하는 값의 타입입니다. 반환 값이 없으면 `Unit`을 사용합니다.

##### 다양한 함수 타입의 예시

| 함수 타입 | 설명 | 이 타입에 맞는 람다 예시 |
| :--- | :--- | :--- |
| `() -> Unit` | 파라미터는 없고, 반환 값도 없음 | `{ println("Hello") }` |
| `(String) -> Int` | `String` 1개를 받아 `Int`를 반환 | `{ it.length }` |
| `(Int, Int) -> String`| `Int` 2개를 받아 `String`을 반환 | `{ a, b -> "Sum is ${a + b}" }` |
| `() -> (Int) -> Int`| 파라미터는 없고, `(Int)->Int` 함수를 반환 | `{ { number -> number * 2 } }`|

-----

#### 함수 타입의 사용

이러한 함수 타입은 코드에서 세 가지 주요한 역할을 합니다.

**1. 변수의 타입으로 선언**
함수를 담는 변수의 타입을 명확히 정의할 수 있습니다.

```kotlin
// calculator라는 변수는 Int 2개를 받아 Int를 반환하는 함수만 담을 수 있습니다.
val calculator: (Int, Int) -> Int

calculator = { a, b -> a + b } // OK
// calculator = { str -> str.length } // 컴파일 오류! 함수 타입이 일치하지 않음
```

**2. 고차 함수의 파라미터 타입으로 선언**
어떤 종류의 함수를 인자로 받을 것인지 명시합니다.

```kotlin
// action 파라미터는 String을 받고 Unit을 반환하는 함수여야 합니다.
fun processString(str: String, action: (String) -> Unit) {
    action(str)
}

processString("hello") { println(it.uppercase()) } // HELLO
```

**3. 고차 함수의 반환 타입으로 선언**
어떤 종류의 함수를 반환할 것인지 명시합니다.

```kotlin
// language 파라미터에 따라 다른 종류의 인사말 생성 함수를 반환
fun getGreetingGenerator(language: String): (String) -> String {
    return when (language) {
        "ko" -> { name -> "안녕하세요, $name 님!" }
        "en" -> { name -> "Hello, $name!" }
        else -> { name -> "Greetings, $name!" }
    }
}

val koreanGreeter = getGreetingGenerator("ko")
println(koreanGreeter("홍길동")) // 안녕하세요, 홍길동 님!
```

-----

#### 널 가능 함수 타입 (Nullable Function Types)

다른 타입과 마찬가지로, 함수 타입 역시 `null`이 될 수 있습니다. 이때는 함수 타입 전체를 괄호로 감싸고 그 뒤에 물음표(`?`)를 붙여야 합니다.

`((Int, Int) -> Int)?`

널 가능 함수 타입의 변수를 호출할 때는 `?.` 안전 호출 연산자와 `invoke()` 메서드를 함께 사용해야 합니다.

```kotlin
var onEvent: (() -> Unit)? = null

// 이벤트가 발생하면 할당된 동작을 수행
fun doSomething() {
    println("어떤 작업을 수행합니다...")
    // onEvent가 null이 아닐 경우에만 함수를 호출(invoke)합니다.
    onEvent?.invoke()
}

onEvent = { println("이벤트 발생!") }
doSomething() // "어떤 작업을 수행합니다..." 와 "이벤트 발생!" 이 모두 출력
```

사실 우리가 `myFunction(a, b)` 형태로 함수를 호출하는 것은 `myFunction.invoke(a, b)`에 대한 문법 설탕입니다. 함수 타입의 변수는 내부적으로 `invoke()`라는 실행 메서드를 가지고 있는 객체인 셈입니다.

함수 타입을 이해하는 것은 고차 함수를 자유자재로 읽고 쓰는 데 필수적입니다. 이는 코틀린이 어떻게 '함수'라는 개념을 타입 시스템 안으로 완벽하게 통합했는지를 보여주는 핵심 문법입니다.