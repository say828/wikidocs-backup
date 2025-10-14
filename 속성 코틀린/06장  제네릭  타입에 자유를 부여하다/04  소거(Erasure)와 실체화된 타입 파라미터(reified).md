### 소거(Erasure)와 실체화된 타입 파라미터(reified)

제네릭은 컴파일 시점에 강력한 타입 검사를 제공하지만, JVM(자바 가상 머신)의 한계로 인해 런타임에는 그 타입 정보가 사라지는 현상이 발생합니다. 이를 \*\*타입 소거(Type Erasure)\*\*라고 합니다.

#### 타입 소거(Type Erasure)란?

코드가 컴파일되어 바이트코드로 변환되는 과정에서, 제네릭 타입 정보가 지워지는 것을 의미합니다. 컴파일러는 `List<String>`과 `List<Int>`가 올바르게 사용되었는지 컴파일 시점에 모두 확인하지만, 일단 컴파일이 끝나고 나면 JVM은 둘 다 그냥 `List`라는 원시 타입(raw type)으로만 인식합니다. 타입 정보가 '지워졌기' 때문입니다.

이러한 타입 소거 때문에, 우리는 런타임에 제네릭 타입을 직접 확인할 수 없습니다.

```kotlin
fun <T> checkType(items: List<Any>) {
    // if (items is List<T>) { ... } // 컴파일 오류!
    // 런타임에는 T가 어떤 타입인지 알 수 없으므로, List<T>인지 검사할 수 없다.
}

fun <T> printClassName() {
    // println(T::class.java.simpleName) // 컴파일 오류!
    // T는 런타임에 실체가 없는 타입이므로 클래스 정보를 가져올 수 없다.
}
```

#### 코틀린의 해법: `reified` 타입 파라미터

대부분의 경우 타입 소거는 문제가 되지 않지만, 때로는 런타임에 제네릭 타입 정보가 반드시 필요한 경우가 있습니다. 코틀린은 이 문제를 해결하기 위해 \*\*`reified`\*\*라는 강력한 기능을 제공합니다.

`reified`는 '구체화된', '실체화된' 이라는 의미로, **타입 소거를 막고 런타임에 제네릭 타입 정보에 접근할 수 있게** 만들어 줍니다.

##### `inline` 함수와의 약속

`reified` 기능은 오직 **`inline` 함수**와 함께 사용될 때만 작동합니다. `inline` 함수는 함수 호출 시점에 함수의 본문 코드를 호출 지점에 그대로 복사해 넣는 방식으로 동작합니다.

이 과정에서, 함수를 호출할 때 사용된 실제 타입(예: `String`)이 코드에 그대로 남게 되므로, 컴파일러는 이 타입 정보를 지우지 않고 런타임까지 유지할 수 있습니다. 즉, 타입이 실체화(reified)되는 것입니다.

##### `reified` 사용하기

이제 `reified`를 사용하여 앞서 실패했던 예제들을 해결해 봅시다.

**1. 런타임에 타입 검사하기**

```kotlin
// 함수를 inline으로 만들고, 타입 파라미터 T에 reified를 붙입니다.
inline fun <reified T> isInstanceOf(value: Any): Boolean {
    // 이제 런타임에 T의 타입을 알 수 있으므로, is T 검사가 가능합니다.
    return value is T
}

fun main() {
    println(isInstanceOf<String>("hello")) // true
    println(isInstanceOf<Int>("hello"))    // false
}
```

**2. 런타임에 타입 정보 활용하기 (실용적인 예제)**
`reified`는 안드로이드의 액티비티 전환이나 JSON 파싱 라이브러리 등에서 매우 유용하게 사용됩니다. 다음은 리스트에서 특정 타입의 첫 번째 요소를 찾아 반환하는 함수 예제입니다.

```kotlin
// List<Any>에 대한 확장 함수로 정의
inline fun <reified T> List<Any>.findFirstInstanceOf(): T? {
    for (item in this) {
        if (item is T) {
            return item // item이 T 타입이면 스마트 캐스팅되어 안전하게 반환
        }
    }
    return null
}

fun main() {
    val mixedList: List<Any> = listOf("Kotlin", 1, "Java", 2.0, Person("Alice", 30))

    // String 타입의 첫 번째 요소를 찾습니다.
    val firstString = mixedList.findFirstInstanceOf<String>()
    println(firstString) // 출력: Kotlin

    // Person 타입의 첫 번째 요소를 찾습니다.
    val firstPerson = mixedList.findFirstInstanceOf<Person>()
    println(firstPerson) // 출력: Person(name=Alice, birthYear=30)
}
```

`inline`과 `reified` 덕분에, 우리는 타입 소거의 한계를 넘어 런타임에도 타입 정보를 온전히 활용하는 강력하고 타입 안전한 제네릭 함수를 만들 수 있습니다.

-----

이것으로 타입에 자유를 부여하는 6장, 제네릭의 여정을 마칩니다. 제네릭은 유연성과 재사용성을 제공하고, 제약은 여기에 안정성을 더하며, 가변성은 타입 계층 구조를 더욱 정교하게 다룰 수 있게 해줍니다. 그리고 `reified`는 JVM의 한계를 뛰어넘는 코틀린만의 특별한 무기입니다. 이 강력한 타입 시스템에 대한 이해를 바탕으로, 다음 Part 3에서는 코틀린 프로그래밍의 또 다른 축인 함수형 프로그래밍의 세계로 떠나보겠습니다.