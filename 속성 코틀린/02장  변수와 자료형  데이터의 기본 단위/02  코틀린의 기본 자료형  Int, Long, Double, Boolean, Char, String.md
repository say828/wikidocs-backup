### 코틀린의 기본 자료형: Int, Long, Double, Boolean, Char, String

변수라는 그릇에 어떤 종류의 데이터를 담을지 명시하는 것을 **자료형(Data Type)** 또는 \*\*타입(Type)\*\*이라고 합니다. 코틀린은 개발에 필요한 대부분의 데이터 종류를 표현할 수 있는 기본적인 자료형들을 풍부하게 내장하고 있습니다. 지금부터 가장 핵심적인 타입들을 하나씩 살펴보겠습니다.

-----

#### 숫자형 (Numbers)

코틀린의 숫자형 타입은 크게 정수형과 실수형으로 나뉩니다.

##### 정수형 (Integer Types)

소수점이 없는 숫자를 다룹니다. 담을 수 있는 값의 크기에 따라 네 가지 타입으로 구분됩니다.

| 타입 | 크기 (Bits) | 표현 가능 범위 |
| :--- | :---: | :--- |
| `Byte` | 8 | -128 \~ 127 |
| `Short` | 16 | -32,768 \~ 32,767 |
| **`Int`** | 32 | 약 -21억 \~ 21억 |
| `Long` | 64 | 매우 큰 수 (약 $-9 \\times 10^{18}$ \~ $9 \\times 10^{18}$) |

  * **`Int`가 기본값:** 코드에서 일반적인 정수를 사용하면, 코틀린은 이를 기본적으로 `Int` 타입으로 간주합니다.
  * **`Long` 타입 명시:** `Int`의 범위를 넘어가는 아주 큰 숫자를 사용하려면, 숫자 뒤에 `L`을 붙여 `Long` 타입임을 명시해야 합니다.

<!-- end list -->

```kotlin
val simpleNumber = 100       // Int 타입으로 추론됨
val bigNumber = 10000000000L // L을 붙여 Long 타입으로 지정
```

##### 실수형 (Floating-Point Types)

소수점이 있는 숫자를 다룹니다. 정밀도에 따라 두 가지 타입으로 나뉩니다.

| 타입 | 크기 (Bits) | 정밀도 |
| :--- | :---: | :--- |
| `Float` | 32 | 소수점 이하 약 6\~7자리 |
| **`Double`** | 64 | 소수점 이하 약 15\~17자리 (더 정밀함) |

  * **`Double`이 기본값:** 코드에서 소수점을 포함한 숫자를 사용하면, 코틀린은 이를 기본적으로 `Double` 타입으로 간주합니다.
  * **`Float` 타입 명시:** `Float` 타입을 사용하려면, 숫자 뒤에 `f` 또는 `F`를 붙여야 합니다.

<!-- end list -->

```kotlin
val pi = 3.141592          // Double 타입으로 추론됨
val piFloat = 3.141592f    // f를 붙여 Float 타입으로 지정
```

-----

#### 논리형 (Boolean)

`Boolean` 타입은 참(`true`) 또는 거짓(`false`), 두 가지 논리적인 값만을 가질 수 있습니다. 주로 조건문(`if`, `when`)이나 반복문(`while`)에서 프로그램의 흐름을 제어하는 데 사용됩니다.

```kotlin
val isUserLoggedIn: Boolean = true
val isReady: Boolean = false
```

-----

#### 문자형 (Char)

`Char` 타입은 단 하나의 문자를 표현합니다. 값은 \*\*작은따옴표(`' '`)\*\*로 감싸야 합니다.

```kotlin
val firstLetter: Char = 'A'
val symbol: Char = '%'
val koreanChar: Char = '가'
```

**주의:** 코틀린의 `Char`는 자바와 달리 숫자로 취급되지 않습니다. `'A' + 1`과 같은 연산은 허용되지 않으며, 문자에 해당하는 숫자 코드를 얻으려면 명시적인 변환이 필요합니다.

-----

#### 문자열 (String)

`String` 타입은 여러 문자의 나열, 즉 문자열을 표현합니다. 값은 \*\*큰따옴표(`" "`)\*\*로 감싸야 합니다. 코틀린의 문자열은 한번 생성되면 내부의 내용을 바꿀 수 없는 **불변(Immutable)** 객체입니다.

```kotlin
val greeting: String = "Hello, World!"
```

코틀린의 `String`은 두 가지 강력하고 편리한 기능을 제공합니다.

##### 문자열 템플릿 (String Templates)

문자열 안에 변수의 값을 손쉽게 포함시킬 수 있습니다. 변수 이름 앞에 `$` 기호를 붙이거나, 복잡한 표현식의 경우 `${...}`로 감싸면 됩니다. 더 이상 `+` 연산자로 문자열을 힘들게 조합할 필요가 없습니다.

```kotlin
val name = "Kotlin"
val message = "My name is $name." // "My name is Kotlin."

val a = 10
val b = 20
println("a + b = ${a + b}") // "a + b = 30"
```

##### 여러 줄 문자열 (Multiline Strings)

\*\*큰따옴표 세 개(`"""..."""`)\*\*를 사용하면, 줄바꿈을 포함하거나 특수문자를 이스케이프(`\n`, `\t` 등)할 필요 없이 보이는 그대로 문자열을 만들 수 있습니다.

```kotlin
val htmlSource = """
    <html>
        <head>
            <title>Hello!</title>
        </head>
        <body>
            <h1>This is a multiline string.</h1>
        </body>
    </html>
""".trimIndent() // trimIndent()는 불필요한 앞쪽 공백을 제거해줍니다.
```

이러한 기본 자료형들은 우리가 다룰 모든 데이터의 기초가 됩니다. 다음으로는 이 타입들을 매번 명시하지 않아도 되는 코틀린의 똑똑한 기능, 타입 추론에 대해 알아보겠습니다.