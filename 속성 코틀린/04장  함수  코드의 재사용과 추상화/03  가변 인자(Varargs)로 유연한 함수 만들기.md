### 가변 인자(Varargs)로 유연한 함수 만들기

함수를 설계하다 보면, "이 함수가 몇 개의 인자를 받을지 미리 알 수 없으면 어떡하지?"라는 고민에 빠질 때가 있습니다. 예를 들어, 전달된 모든 숫자의 합계를 구하는 `sum` 함수를 만든다고 생각해 봅시다. 두 수를 더하는 `sum(a: Int, b: Int)`, 세 수를 더하는 `sum(a: Int, b: Int, c: Int)` 처럼 모든 경우의 수에 맞춰 함수를 오버로딩하는 것은 비효율적이고 불가능에 가깝습니다.

이러한 문제를 해결하기 위해 코틀린은 **가변 인자(Variable number of arguments)**, 즉 **`vararg`** 라는 기능을 제공합니다. `vararg`를 사용하면 함수가 0개 이상의 인자를 유연하게 받아들일 수 있습니다.

#### `vararg` 사용법

파라미터 타입 앞에 `vararg` 키워드를 붙여주기만 하면 됩니다.

```kotlin
fun printNumbers(vararg numbers: Int) {
    println("전달받은 숫자의 개수: ${numbers.size}")
    for (number in numbers) {
        print("$number ")
    }
    println()
}
```

**`vararg` 파라미터의 내부 동작:**
`vararg`로 선언된 파라미터는 함수 내부로 전달될 때 **배열(Array)** 로 취급됩니다. 위 예시에서 `numbers` 파라미터의 타입은 `Array<Int>` 입니다. 따라서 우리는 배열의 `size` 프로퍼티를 사용하거나 `for` 루프를 통해 각 요소에 접근할 수 있습니다.

이제 이 함수를 다양한 개수의 인자로 호출해 봅시다.

```kotlin
fun main() {
    printNumbers(1, 2, 3) 
    // 출력:
    // 전달받은 숫자의 개수: 3
    // 1 2 3 

    printNumbers(10, 20, 30, 40, 50)
    // 출력:
    // 전달받은 숫자의 개수: 5
    // 10 20 30 40 50 

    printNumbers() // 인자를 하나도 전달하지 않아도 됩니다.
    // 출력:
    // 전달받은 숫자의 개수: 0
    // 
}
```

-----

#### `vararg` 사용 규칙

`vararg`는 매우 편리하지만, 몇 가지 지켜야 할 규칙이 있습니다.

1.  **하나의 함수에는 하나의 `vararg` 파라미터만 허용됩니다.**

    ```kotlin
    // fun invalidFunc(vararg a: Int, vararg b: String) // 컴파일 오류!
    ```

2.  **`vararg`가 마지막 파라미터가 아닐 경우, 그 뒤에 오는 인자는 반드시 이름 있는 인자로 전달해야 합니다.**
    컴파일러가 어디까지가 `vararg`이고 어디부터가 다음 파라미터인지 구분할 수 있도록 도와주기 위함입니다.

    ```kotlin
    fun printUserInfo(vararg skills: String, department: String) {
        println("부서: $department")
        println("보유 기술: ${skills.joinToString(", ")}")
    }

    // 호출 시
    printUserInfo("Java", "Kotlin", "Spring", department = "개발1팀") // 올바른 호출
    // printUserInfo("Java", "Kotlin", "개발1팀") // 컴파일 오류!
    ```

-----

#### 스프레드 연산자(Spread Operator): `*`

만약 함수에 전달할 값들이 이미 배열에 담겨 있다면 어떻게 해야 할까요? 이 배열을 `vararg` 함수에 그대로 전달할 수는 없습니다.

```kotlin
val numbersArray = intArrayOf(1, 2, 3, 4, 5)
// printNumbers(numbersArray) // 컴파일 오류! Int와 IntArray는 타입이 다릅니다.
```

이때 사용하는 것이 바로 **스프레드 연산자(`*`)** 입니다. 스프레드 연산자는 배열 앞에 붙여서, **배열의 내용을 펼쳐서** 각각의 요소를 개별 인자로 전달해주는 역할을 합니다.

```kotlin
val numbersArray = intArrayOf(1, 2, 3, 4, 5)

// *numbersArray는 1, 2, 3, 4, 5 로 펼쳐져서 전달됩니다.
printNumbers(*numbersArray)
// 출력:
// 전달받은 숫자의 개수: 5
// 1 2 3 4 5
```

마치 배열이라는 상자를 열어 그 안의 내용물들을 하나씩 꺼내어 함수에 전달하는 것과 같습니다.

`vararg`는 로깅 함수, 문자열 포매팅, 여러 값에 대한 연산 등 인자의 개수가 유동적인 유틸리티 함수를 만들 때 매우 유용하며, API를 훨씬 더 유연하고 사용하기 쉽게 만들어 줍니다.