### when: 자바의 switch를 뛰어넘는 강력함

`if-else`가 두세 가지 갈림길에서 선택을 내리는 도구라면, \*\*`when`\*\*은 수많은 갈림길이 있는 복잡한 교차로에서 길을 안내하는 능숙한 교통경찰과 같습니다. 자바의 `switch` 문을 써본 개발자라면, `break`를 잊어서 발생하는 버그나 제한적인 사용성에 답답함을 느낀 적이 있을 것입니다. 코틀린의 `when`은 `switch`를 단순히 대체하는 것을 넘어, 비교할 수 없을 정도로 강력하고 유연하며 안전한 기능을 제공합니다.

#### 기본 사용법과 `switch`와의 차이점

`when`의 가장 기본적인 형태는 `switch`와 비슷합니다. 괄호 안의 값과 일치하는 분기(`branch`)를 찾아 실행합니다.

```kotlin
val dayOfWeek = 3

when (dayOfWeek) {
    1 -> println("월요일")
    2 -> println("화요일")
    3 -> println("수요일") // dayOfWeek가 3이므로 이 코드가 실행됩니다.
    4 -> println("목요일")
    5 -> println("금요일")
    else -> println("주말 또는 잘못된 값") // default와 같은 역할
}
```

하지만 벌써 중요한 차이점이 보입니다.

1.  **`break`가 필요 없다:** `when`은 일치하는 하나의 분기만 실행하고 즉시 `when` 블록을 빠져나갑니다. `break`를 실수로 빠뜨려 원치 않는 코드가 실행되는 'fall-through' 버그가 원천적으로 불가능합니다.
2.  **`else`의 존재:** `switch`의 `default`와 비슷한 `else` 분기는 다양한 상황을 처리하며, 특히 `when`을 표현식으로 사용할 때 코드의 안정성을 보장하는 중요한 역할을 합니다.

-----

#### `when`의 다채로운 능력

`when`의 진정한 힘은 단순 값 비교를 넘어선 다채로운 조건 처리 능력에서 나옵니다.

##### 1\. 여러 조건 한번에 처리하기 (쉼표 사용)

쉼표(`,`)를 이용해 여러 값을 하나의 분기에서 처리할 수 있습니다.

```kotlin
val dayOfWeek = 7
when (dayOfWeek) {
    1, 2, 3, 4, 5 -> println("주중 (Weekday)")
    6, 7 -> println("주말 (Weekend)") // dayOfWeek가 7이므로 여기가 실행됩니다.
}
```

##### 2\. 범위(Range) 사용하기 (`in` 연산자)

`in` 연산자와 함께 범위를 지정하여 값이 특정 구간에 포함되는지 검사할 수 있습니다. 점수에 따라 등급을 매기는 로직에 매우 유용합니다.

```kotlin
val score = 85
when (score) {
    in 90..100 -> println("A 등급")
    in 80..89 -> println("B 등급") // 85는 이 범위에 속합니다.
    in 70..79 -> println("C 등급")
    else -> println("F 등급")
}
```

##### 3\. 타입 검사하기 (`is` 연산자)

`is` 연산자와 결합하면, `when`은 변수의 타입을 검사하고 스마트 캐스팅까지 완벽하게 수행하는 멋진 도구가 됩니다.

```kotlin
fun checkType(obj: Any) { // Any는 코틀린의 최상위 타입입니다.
    when (obj) {
        is String -> {
            // obj가 String 타입임이 확인되고, String으로 스마트 캐스팅됩니다.
            println("문자열 길이: ${obj.length}")
        }
        is Int -> {
            println("정수 값의 두 배: ${obj * 2}")
        }
        else -> {
            println("알 수 없는 타입")
        }
    }
}
```

##### 4\. 인자 없는 `when` (if-else if 대체)

`when`의 괄호 안에 아무 인자도 넣지 않으면, 각 분기의 조건부를 `true` 또는 `false`를 반환하는 일반 조건식으로 사용할 수 있습니다. 이는 복잡한 `if-else if` 체인을 훨씬 더 깔끔하게 만들어 줍니다.

```kotlin
val name = "Kotlin"
val score = 95

when {
    name == "Kotlin" && score > 90 -> {
        println("당신은 코틀린 고수군요!")
    }
    name.startsWith("J") -> {
        println("혹시 Java 개발자이신가요?")
    }
    else -> {
        println("다양한 언어를 공부하는군요!")
    }
}
```

-----

#### `when`을 표현식(Expression)으로 사용하기

`if`와 마찬가지로, `when` 역시 값을 반환하는 표현식으로 사용할 수 있습니다. 이는 코드를 매우 간결하고 선언적으로 만들어주는 핵심 기능입니다.

```kotlin
val score = 85
val grade = when (score) {
    in 90..100 -> "A"
    in 80..89 -> "B" // 이 분기의 값 "B"가 grade 변수에 할당됩니다.
    in 70..79 -> "C"
    else -> "F"
}
println("당신의 학점은 ${grade}입니다.")
```

**중요:** `when`을 표현식으로 사용할 때는, 컴파일러가 모든 가능한 경우를 처리했다고 확신할 수 있어야 합니다. 따라서 `else` 분기는 **반드시 포함되어야 합니다.** 이는 어떤 상황에서도 값이 반드시 할당되도록 보장하여 런타임 오류를 방지하는 코틀린의 안전장치입니다.

`when`은 단순한 조건 분기문을 넘어, 값, 범위, 타입 등 다양한 패턴을 검사하고 그 결과를 값으로 반환하는 **'패턴 매칭(Pattern Matching)'** 의 초기 형태를 보여줍니다. 이 강력한 도구를 잘 활용하는 것만으로도 당신의 코드는 한 차원 높은 수준의 가독성과 안정성을 갖추게 될 것입니다.