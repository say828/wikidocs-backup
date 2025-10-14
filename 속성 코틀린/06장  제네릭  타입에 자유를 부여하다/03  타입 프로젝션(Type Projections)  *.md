### 타입 프로젝션(Type Projections): \*

제네릭 가변성은 클래스를 설계할 때 `in`과 `out`을 통해 데이터 흐름을 명시하는 강력한 도구입니다. 하지만 만약 이미 만들어진 클래스(특히 `in`이나 `out`이 붙지 않은 무공변 클래스)를 사용하는 입장에서, 특정 함수에 한해 일시적으로 공변성이나 반공변성을 적용하고 싶다면 어떻게 해야 할까요?

예를 들어, `MutableList<T>`는 데이터를 읽고 쓸 수 있으므로 무공변입니다. 따라서 `MutableList<String>`은 `MutableList<Any>`와 아무 관계가 없습니다.

```kotlin
fun printAnyList(list: MutableList<Any>) {
    list.forEach { println(it) }
}

val strings: MutableList<String> = mutableListOf("A", "B", "C")
// printAnyList(strings) // 컴파일 오류! MutableList<String>은 MutableList<Any>의 하위 타입이 아님
```

`printAnyList` 함수는 리스트의 내용을 읽기만 할 뿐인데도, 무공변성 때문에 `MutableList<String>`을 인자로 받을 수 없습니다. 이 문제를 해결하기 위해 사용하는 것이 바로 \*\*타입 프로젝션(Type Projection)\*\*입니다.

-----

#### 스타 프로젝션 (Star-Projection): `*`

타입 프로젝션의 가장 간단한 형태로 \*\*스타 프로젝션(`*`)\*\*이 있습니다. 이는 제네릭 타입의 타입 파라미터에 대해 \*\*"나는 이 타입이 무엇인지 정확히는 모르지만, 어떤 특정 타입 하나로 정해져 있다"\*\*고 컴파일러에게 알려주는 것입니다.

`List<*>`는 `List<Any>`와 다릅니다.

  * **`List<Any>`**: `Int`, `String`, `Person` 등 **아무 타입의 값이나 섞어서** 담을 수 있는 리스트입니다.
  * **`List<*>`**: \*\*어떤 특정 타입 `T`에 대한 `List<T>`\*\*를 의미합니다. 그 `T`가 `String`일 수도, `Int`일 수도 있지만, 리스트 내부의 모든 요소는 그 한 가지 타입으로 통일되어 있습니다.

##### 스타 프로젝션의 안전 규칙

컴파일러는 `*`로 지정된 타입 `T`가 무엇인지 정확히 모르기 때문에, 타입 안전성을 보장하기 위해 다음과 같은 엄격한 규칙을 적용합니다.

1.  **읽기(out): 안전함.** 값을 꺼내는 것은 안전합니다. 어떤 타입이 나올지는 모르지만, 코틀린의 모든 타입은 `Any?`의 하위 타입이므로, 꺼낸 값은 `Any?` 타입으로 간주됩니다.
2.  **쓰기(in): 불가능.** 값을 넣는 것은 안전하지 않으므로 금지됩니다. 만약 `MutableList<*>`가 실제로는 `MutableList<String>`을 참조하고 있는데, 여기에 `Int` 값을 넣으려는 시도를 막아야 하기 때문입니다.

이제 스타 프로젝션을 사용하여 `printAnyList` 문제를 해결해 봅시다.

```kotlin
// 파라미터 타입을 MutableList<*>로 변경
fun printAnyList(list: MutableList<*>) {
    // 1. 읽기(out): 가능. 각 item은 Any? 타입으로 취급됨
    for (item in list) {
        println(item)
    }

    // 2. 쓰기(in): 불가능. 어떤 타입을 넣어야 할지 알 수 없으므로 컴파일 오류 발생
    // list.add(10) // 컴파일 오류!
    
    // size와 같이 타입 T와 무관한 메서드는 호출 가능
    println("리스트 크기: ${list.size}") 
}

fun main() {
    val strings: MutableList<String> = mutableListOf("A", "B", "C")
    val ints: MutableList<Int> = mutableListOf(1, 2, 3)

    printAnyList(strings) // OK!
    printAnyList(ints)    // OK!
}
```

`MutableList<*>`를 사용함으로써, `printAnyList` 함수는 이제 어떤 타입의 `MutableList`든 안전하게 인자로 받아 그 내용을 출력할 수 있게 되었습니다.

#### 정리

  * **타입 프로젝션**은 이미 선언된 제네릭 클래스를 사용하는 입장에서, 특정 사용처에 한해 가변성을 부여하는 기술입니다.
  * \*\*스타 프로젝션 (`*`)\*\*은 "정확한 타입은 모르지만, 어떤 한 가지 특정 타입으로 채워진 제네릭 타입"임을 명시할 때 사용합니다.
  * `T`가 `*`로 프로젝션되면, 해당 객체에서 `T`를 반환하는 메서드는 `Any?`를 반환하는 것으로 취급되고, `T`를 파라미터로 받는 메서드는 호출이 금지됩니다.

이는 제네릭을 다루는 API를 작성할 때, 타입에 대해 알 필요가 없는 유틸리티 함수(예: `printAll`, `clear`, `getSize` 등)를 매우 유연하게 만들 수 있도록 도와주는 실용적인 도구입니다.