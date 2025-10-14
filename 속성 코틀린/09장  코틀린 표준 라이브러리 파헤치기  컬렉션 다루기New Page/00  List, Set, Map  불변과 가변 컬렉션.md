### List, Set, Map: 불변과 가변 컬렉션

코틀린 컬렉션 프레임워크의 가장 중요한 설계 철학은 **읽기 전용(Read-only) 컬렉션**과 **변경 가능(Mutable) 컬렉션**의 인터페이스를 명확하게 분리한 것입니다.

  * **읽기 전용 컬렉션 (`List`, `Set`, `Map`)**: 한 번 생성되면 요소의 추가, 삭제, 변경이 불가능한 컬렉션입니다. 📖 마치 출판된 책처럼 내용을 읽을 수만 있습니다.
  * **변경 가능 컬렉션 (`MutableList`, `MutableSet`, `MutableMap`)**: 생성된 이후에도 자유롭게 요소를 추가, 삭제, 변경할 수 있는 컬렉션입니다. 📓 계속해서 내용을 고쳐 쓸 수 있는 노트와 같습니다.

이러한 분리는 "기본적으로 불변성을 지향하라"는 코틀린의 철학과 맞닿아 있습니다. 변경이 꼭 필요하지 않은 데이터는 읽기 전용 컬렉션에 담아둠으로써, 의도치 않은 데이터 변경으로 인한 버그를 원천적으로 방지하고 코드의 안정성을 높일 수 있습니다.

-----

#### 1\. List: 순서가 있는 컬렉션

\*\*`List`\*\*는 가장 흔하게 사용되는 컬렉션으로, **데이터가 저장된 순서를 기억**하며 인덱스(index)를 통해 각 요소에 접근할 수 있습니다. 요소의 중복을 허용합니다.

##### 읽기 전용 `List`

`listOf()` 함수를 사용하여 생성합니다. `add()`나 `remove()` 같은 수정 관련 메서드가 아예 존재하지 않습니다.

```kotlin
val numbers: List<Int> = listOf(1, 2, 3, 4, 3) // 중복 허용
println(numbers[0])      // 1
println(numbers.size)    // 5
// numbers.add(5) // 컴파일 오류! 읽기 전용 리스트는 수정할 수 없습니다.
```

##### 변경 가능 `MutableList`

`mutableListOf()` 함수를 사용하여 생성합니다. `add()`, `remove()` 등 다양한 수정 메서드를 제공합니다.

```kotlin
val mutableNumbers: MutableList<Int> = mutableListOf(1, 2, 3)
println(mutableNumbers) // [1, 2, 3]

mutableNumbers.add(4)      // 맨 끝에 4 추가
mutableNumbers.removeAt(0) // 0번 인덱스 요소(1) 제거
mutableNumbers[1] = 100    // 1번 인덱스 요소(3)를 100으로 변경

println(mutableNumbers) // [2, 100, 4]
```

-----

#### 2\. Set: 순서가 없고 중복을 허용하지 않는 컬렉션

\*\*`Set`\*\*은 **요소의 중복을 허용하지 않는** 특별한 컬렉션입니다. 저장된 요소들의 순서는 보장되지 않습니다.

##### 읽기 전용 `Set`

`setOf()` 함수로 생성하며, 중복된 요소는 자동으로 제거됩니다.

```kotlin
val fruits: Set<String> = setOf("사과", "바나나", "오렌지", "바나나")
println(fruits) // [사과, 바나나, 오렌지] (순서는 다를 수 있음)
```

##### 변경 가능 `MutableSet`

`mutableSetOf()` 함수로 생성하며, `add()`, `remove()`로 요소를 관리할 수 있습니다.

```kotlin
val mutableFruits: MutableSet<String> = mutableSetOf("사과")
mutableFruits.add("바나나") // true 반환 (추가 성공)
mutableFruits.add("사과")   // false 반환 (이미 존재하므로 추가 실패)
println(mutableFruits) // [사과, 바나나]
```

-----

#### 3\. Map: 키-값(Key-Value) 쌍의 컬렉션

\*\*`Map`\*\*은 \*\*고유한 키(Key)\*\*와 그에 대응하는 \*\*값(Value)\*\*을 쌍으로 묶어 저장하는 컬렉션입니다. '사전'이나 '전화번호부'처럼 키를 통해 값을 빠르게 찾아올 수 있습니다.

##### 읽기 전용 `Map`

`mapOf()` 함수와 `to` 중위 함수를 사용하여 생성합니다.

```kotlin
val userAges: Map<String, Int> = mapOf(
    "Alice" to 30,
    "Bob" to 25
)
println(userAges["Alice"]) // 30
// userAges["Charlie"] = 40 // 컴파일 오류!
```

##### 변경 가능 `MutableMap`

`mutableMapOf()` 함수로 생성하며, `[]` 문법이나 `put()` 메서드로 요소를 추가/수정할 수 있습니다.

```kotlin
val mutableUserAges: MutableMap<String, Int> = mutableMapOf("Alice" to 30)
mutableUserAges["Bob"] = 25      // 새로운 쌍 추가
mutableUserAges["Alice"] = 31    // "Alice" 키의 값 수정
mutableUserAges.remove("Bob")    // "Bob" 키의 쌍 제거
println(mutableUserAges) // {Alice=31}
```

**컬렉션 사용의 원칙:** 프로그램을 설계할 때, 생성 후에 변경할 필요가 없다면 항상 **읽기 전용 컬렉션**을 우선적으로 사용하세요. 이는 당신의 코드를 훨씬 더 안전하고 예측 가능하게 만들어 줄 것입니다.