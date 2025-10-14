### forEach, map, filter, flatMap, reduce, fold 등 핵심 연산

8장에서 배운 람다와 고차 함수가 코틀린 표준 라이브러리의 컬렉션 API와 만날 때, 데이터 처리는 `for` 루프를 사용하던 명령형(Imperative) 스타일에서 벗어나, 훨씬 더 간결하고 우아한 선언형(Declarative) 스타일로 진화합니다.

선언형 스타일은 '어떻게' 처리할지를 단계별로 지시하는 대신, '무엇을' 원하는지를 서술하는 방식입니다. 이제부터 데이터 처리를 위한 가장 핵심적인 연산들을 하나씩 정복해 봅시다.

-----

#### `forEach`: 각 요소에 대해 작업 수행

`forEach`는 `for` 루프의 가장 직접적인 함수형 대체재입니다. 컬렉션의 각 요소를 순회하며 주어진 람다를 실행합니다.

```kotlin
val fruits = listOf("사과", "바나나", "딸기")

// for 루프 사용
for (fruit in fruits) {
    println("${fruit}!")
}

// forEach 사용
fruits.forEach { fruit -> println("${fruit}!") }

// 'it'을 사용하면 더 간결해집니다.
fruits.forEach { println("${it}!") }
```

-----

#### `map`: 각 요소를 변환하기

\*\*`map`\*\*은 컬렉션의 **각 요소를 주어진 람다에 따라 변환**하여, 그 결과들을 모아 **새로운 리스트를 생성**하는 매우 중요한 함수입니다. 원본 컬렉션은 변경되지 않으며, 결과 리스트의 크기는 항상 원본과 같습니다.

🏭 **비유:** `map`은 각 재료(요소)가 컨베이어 벨트를 지나가며 특정 가공(람다)을 거쳐 새로운 제품(변환된 요소)으로 탄생하는 공장 라인과 같습니다.

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

// 각 숫자를 제곱한 새로운 리스트 만들기
val squaredNumbers = numbers.map { it * it }
println(squaredNumbers) // 출력: [1, 4, 9, 16, 25]

// 각 과일의 이름 길이를 담은 리스트 만들기
val nameLengths = fruits.map { it.length }
println(nameLengths) // 출력: [2, 3, 2]
```

-----

#### `filter`: 조건에 맞는 요소만 걸러내기

\*\*`filter`\*\*는 컬렉션의 각 요소를 검사하여, **람다의 결과가 `true`인 요소들만**을 모아 **새로운 리스트를 생성**합니다.

🕵️‍♀️ **비유:** `filter`는 특정 기준(람다)을 통과한 우량품만 골라내는 품질 검사관과 같습니다.

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8)

// 짝수만 걸러내기
val evenNumbers = numbers.filter { it % 2 == 0 }
println(evenNumbers) // 출력: [2, 4, 6, 8]

// 이름이 3글자 이상인 과일만 걸러내기
val longNameFruits = fruits.filter { it.length >= 3 }
println(longNameFruits) // 출력: [바나나, 딸기]
```

-----

#### `flatMap`: 변환 후 펼치기

\*\*`flatMap`\*\*은 `map`과 `flatten`(중첩된 컬렉션을 하나로 펼치는 기능)을 합친 연산입니다. 각 요소를 변환하되, 그 **변환 결과가 컬렉션**이어야 합니다. `flatMap`은 이렇게 만들어진 여러 개의 내부 컬렉션들을 모두 합쳐 **하나의 단일 리스트로 펼쳐서** 반환합니다.

```kotlin
val sentences = listOf("코틀린은 즐거워", "함수형 프로그래밍")

// map: 결과가 List<List<String>> 형태의 중첩 리스트가 됨
val wordsByMap = sentences.map { it.split(" ") }
println(wordsByMap) // 출력: [[코틀린은, 즐거워], [함수형, 프로그래밍]]

// flatMap: 모든 단어를 하나의 List<String>으로 펼쳐줌
val allWords = sentences.flatMap { it.split(" ") }
println(allWords) // 출력: [코틀린은, 즐거워, 함수형, 프로그래밍]
```

-----

#### `reduce` 와 `fold`: 모든 요소를 하나로 합치기

`reduce`와 `fold`는 컬렉션의 모든 요소를 순회하며 연산을 적용하여 **하나의 최종 결과값**을 만들어내는 집계(aggregate) 함수입니다.

##### `reduce`

첫 번째 요소를 초기값으로 사용하여, `(누적값, 현재 요소) -> 새로운 누적값` 형태의 람다를 순차적으로 적용합니다. 컬렉션이 비어있으면 예외가 발생합니다.

```kotlin
val numbers = listOf(1, 2, 3, 4, 5) // 합계: 15

val sumByReduce = numbers.reduce { accumulator, current ->
    println("$accumulator + $current")
    accumulator + current 
}
println(sumByReduce) // 출력: 15
```

##### `fold`

`reduce`와 유사하지만, **별도의 초기값**을 제공한다는 점이 다릅니다. 따라서 빈 컬렉션에도 안전하게 사용할 수 있으며(초기값 반환), 초기값의 타입에 따라 결과값의 타입을 유연하게 정할 수 있습니다.

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

// 초기값을 100으로 시작하여 모든 요소를 더함
val sumByFold = numbers.fold(100) { accumulator, current ->
    accumulator + current
}
println(sumByFold) // 출력: 115
```

이러한 핵심 연산들을 조합(chaining)하면, `numbers.filter { it > 3 }.map { it * 2 }` 처럼 복잡한 데이터 처리 파이프라인을 매우 간결하고 가독성 높게 표현할 수 있습니다.