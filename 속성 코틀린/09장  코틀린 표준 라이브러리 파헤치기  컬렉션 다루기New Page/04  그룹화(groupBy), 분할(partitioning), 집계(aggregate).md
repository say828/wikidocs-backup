### 그룹화(groupBy), 분할(partitioning), 집계(aggregate)

`map`과 `filter`가 컬렉션의 각 요소를 변환하거나 선택하는 '미시적인' 연산이었다면, 지금부터 배울 함수들은 컬렉션 전체를 조망하며 데이터를 더 의미 있는 구조로 재편성하거나 요약하는 '거시적인' 연산입니다.

-----

#### 그룹화 (Grouping): `groupBy`

\*\*`groupBy`\*\*는 컬렉션의 각 요소를 특정 기준(key)에 따라 여러 그룹으로 묶어 \*\*`Map`\*\*을 생성하는 강력한 함수입니다. 람다는 각 요소를 어떤 키에 할당할지를 결정하며, 생성된 `Map`의 값(value)은 해당 키를 가진 요소들의 `List`가 됩니다.

📮 **비유:** `groupBy`는 우체국의 우편물 분류기와 같습니다. 각 편지(요소)에 적힌 우편번호(키)를 보고, 동일한 우편번호를 가진 편지들을 같은 우편 자루(`List`)에 담는 것과 같습니다.

##### 예시: 도시별로 사람 묶기

```kotlin
data class Person(val name: String, val city: String)

val people = listOf(
    Person("Alice", "서울"),
    Person("Bob", "부산"),
    Person("Charlie", "서울"),
    Person("David", "부산"),
    Person("Eve", "제주")
)

// 거주 도시(city)를 기준으로 사람들을 그룹화합니다.
val peopleByCity: Map<String, List<Person>> = people.groupBy { it.city }

// pretty print the map
peopleByCity.forEach { (city, residents) ->
    println("[$city]")
    residents.forEach { person ->
        println(" - ${person.name}")
    }
}
```

**실행 결과:**

```
[서울]
 - Alice
 - Charlie
[부산]
 - Bob
 - David
[제주]
 - Eve
```

`groupBy` 하나만으로 복잡한 반복문과 `Map` 생성 로직 없이도 데이터를 손쉽게 분류할 수 있습니다.

-----

#### 분할 (Partitioning): `partition`

\*\*`partition`\*\*은 `groupBy`의 특별한 경우로, 컬렉션을 **단 두 개의 그룹으로 나누는** 데 사용됩니다. 람다는 각 요소를 검사하여 `true` 또는 `false`를 반환해야 하며, `partition`은 그 결과에 따라 **두 개의 리스트가 담긴 `Pair` 객체**를 반환합니다. 첫 번째 리스트는 조건을 만족하는(`true`) 요소들을, 두 번째 리스트는 만족하지 못하는(`false`) 요소들을 담습니다.

🪙 **비유:** `partition`은 동전을 두 종류로 나누는 것과 같습니다. '이 동전은 100원짜리인가?'라는 기준(람다)에 따라 '100원짜리 더미'와 '나머지 더미' 두 개로 완벽하게 분리합니다.

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6)

// 숫자를 짝수와 홀수 그룹으로 분할합니다.
val (evens, odds) = numbers.partition { it % 2 == 0 }

println("짝수: $evens") // 출력: 짝수: [2, 4, 6]
println("홀수: $odds") // 출력: 홀수: [1, 3, 5]
```

`Pair`를 반환하는 특징 덕분에, 위 예시처럼 구조 분해 선언 `val (evens, odds) = ...` 을 사용하면 결과를 두 개의 변수에 바로 할당하여 매우 깔끔한 코드를 작성할 수 있습니다.

-----

#### 집계 (Aggregating): 데이터 요약하기

컬렉션 전체의 통계적인 값을 계산하는 다양한 집계 함수들이 있습니다.

  * **`count()`**: 요소의 개수를 반환합니다. 람다를 전달하면, 조건을 만족하는 요소의 개수만 셉니다.
    ```kotlin
    val countOfEvens = numbers.count { it % 2 == 0 } // 3
    ```
  * **`sum()`**: 숫자 타입 컬렉션의 모든 요소의 합계를 반환합니다. (Kotlin 1.4부터는 더 유연한 `sumOf()`가 추가되었습니다.)
    ```kotlin
    val total = numbers.sum() // 21
    ```
  * **`average()`**: 숫자 타입 컬렉션의 평균값을 `Double`로 반환합니다.
    ```kotlin
    val average = numbers.average() // 3.5
    ```
  * **`minOrNull()` & `maxOrNull()`**: 컬렉션의 최소값 또는 최대값을 반환합니다. 컬렉션이 비어있을 경우 예외 대신 `null`을 반환하기 때문에 이름에 `OrNull`이 붙습니다.
    ```kotlin
    println(numbers.maxOrNull()) // 6
    println(emptyList<Int>().minOrNull()) // null
    ```

이러한 함수들은 데이터를 단순히 변환하는 것을 넘어, 데이터로부터 의미 있는 통찰을 얻어내는 데이터 분석의 첫걸음입니다.