### 타입 파라미터 제약(Generic Constraints)

제네릭은 어떤 타입이든 받을 수 있어 매우 유연하지만, 때로는 이러한 '완전한 자유'가 문제가 될 수 있습니다. 만약 제네릭 함수나 클래스 내부에서 해당 타입이 특정 기능(메서드)을 가지고 있다고 가정하고 코드를 작성해야 한다면 어떻게 될까요?

예를 들어, 두 개의 값을 비교해서 더 큰 값을 반환하는 제네릭 함수 `findMax()`를 만든다고 상상해 봅시다.

```kotlin
fun <T> findMax(a: T, b: T): T {
    // if (a > b) { ... } // 컴파일 오류!
    // ...
}
```

위 코드는 `a > b` 부분에서 컴파일 오류가 발생합니다. 왜냐하면 타입 파라미터 `T`는 `Any?` 타입, 즉 어떤 타입이든 될 수 있기 때문입니다. 컴파일러 입장에서는 `T`가 숫자처럼 크기 비교가 가능한 타입이라는 보장이 전혀 없으므로, `>` 연산자를 사용할 수 없다고 판단하는 것입니다.

이러한 문제를 해결하기 위해 코틀린은 **타입 파라미터 제약(Generic Constraints)** 기능을 제공합니다. 이를 통해 제네릭 타입 `T`가 받을 수 있는 타입의 종류를 특정 슈퍼클래스나 인터페이스로 제한할 수 있습니다.

#### 상한 경계 (Upper Bound) 설정

가장 일반적인 제약 조건은 \*\*상한 경계(Upper Bound)\*\*를 설정하는 것입니다. 타입 파라미터 뒤에 콜론(`:`)을 붙이고 상위 타입을 명시하면, 해당 타입 파라미터에는 **지정된 상위 타입 또는 그 하위 타입**만 올 수 있습니다.

`<T : SuperType>`

이제 이 제약을 사용하여 `findMax` 함수를 완성해 봅시다. 값의 크기를 비교하는 기능은 표준 라이브러리의 `Comparable<T>` 인터페이스에 정의되어 있습니다. 따라서 타입 `T`가 `Comparable<T>`를 구현하도록 제약을 걸면 됩니다.

```kotlin
// T는 Comparable<T> 인터페이스를 구현하는 타입만 받을 수 있습니다.
fun <T : Comparable<T>> findMax(a: T, b: T): T {
    // 이제 컴파일러는 T가 비교 가능한(comparable) 타입임을 알기 때문에
    // > 연산자를 안전하게 사용할 수 있습니다.
    return if (a > b) a else b
}

fun main() {
    println(findMax(10, 20))      // 출력: 20 (Int는 Comparable<Int>를 구현)
    println(findMax("Kotlin", "Java")) // 출력: Kotlin (String은 Comparable<String>를 구현)
    
    // class User // Comparable을 구현하지 않은 클래스
    // val user1 = User()
    // val user2 = User()
    // findMax(user1, user2) // 컴파일 오류! User는 Comparable 타입이 아닙니다.
}
```

이제 `findMax` 함수는 타입 안전성을 완벽하게 보장합니다. `Comparable`을 구현하지 않은 타입을 인자로 전달하려는 시도는 컴파일 시점에 차단됩니다.

-----

#### 여러 제약 조건 설정: `where` 절

타입 파라미터 `T`가 여러 인터페이스를 구현하거나 특정 클래스를 상속받으면서 동시에 특정 인터페이스를 구현해야 하는 등, **여러 개의 제약 조건을 동시에** 걸고 싶을 때는 `where` 절을 사용합니다.

예를 들어, 어떤 값을 화면에 출력할 수 있으면서(`Appendable`), 동시에 닫을 수 있는(`Closeable`) 타입만 처리하는 함수를 만들어 봅시다.

```kotlin
// T는 Appendable과 Closeable을 모두 구현해야 합니다.
fun <T> useAndClose(item: T) where T : Appendable, T : java.io.Closeable {
    try {
        item.append("Hello, World!")
        println(item)
    } finally {
        item.close()
    }
}
```

`where` 절을 사용하면 함수 시그니처 뒤에 여러 제약 조건을 깔끔하게 나열할 수 있습니다.

타입 파라미터 제약은 제네릭의 유연함에 '안전벨트'를 매주는 것과 같습니다. 이를 통해 "나는 어떤 타입 `T`든 받겠지만, 그 타입은 최소한 이러이러한 기능은 할 수 있어야 해"라고 컴파일러에게 명확히 알려줄 수 있습니다. 이는 재사용 가능하면서도 훨씬 더 강력하고 안정적인 제네릭 코드를 작성할 수 있게 해주는 핵심적인 기능입니다.