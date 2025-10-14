### 타입 별칭(Type Aliases)

코드를 작성하다 보면, 특히 제네릭이나 함수 타입을 사용할 때, 타입 이름이 굉장히 길고 복잡해지는 경우가 있습니다.

```kotlin
// 유저 데이터를 담는 맵. 타입이 길고 복잡하다.
val userMap: MutableMap<String, List<Pair<Product, Int>>> = mutableMapOf()

// 클릭 이벤트를 처리하는 람다. 함수 타입이 길다.
fun setOnClickListener(listener: (View, MotionEvent) -> Boolean) { /* ... */ }
```

이처럼 긴 타입 선언은 코드를 읽기 어렵게 만들고, 반복해서 사용하기에도 번거롭습니다.

\*\*타입 별칭(Type Alias)\*\*은 `typealias` 키워드를 사용하여, 기존의 복잡한 타입에 \*\*더 짧거나 의미 있는 새 이름(별칭)\*\*을 붙여주는 기능입니다.

-----

#### `typealias` 사용법

`typealias`는 새로운 타입을 만드는 것이 아니라, 단순히 기존 타입에 다른 이름을 부여하는 것입니다. 컴파일러는 이 별칭을 실제 원래 타입으로 대체하여 처리합니다.

**기본 구조: `typealias 별칭이름 = 원래타입`**

위의 복잡한 예제들을 `typealias`를 사용하여 개선해 봅시다.

##### 복잡한 제네릭 타입에 별칭 붙이기

```kotlin
// Product와 수량을 담는 Pair에 대한 별칭
typealias CartItem = Pair<Product, Int>
// 사용자 ID를 키로, 장바구니 목록을 값으로 갖는 맵에 대한 별칭
typealias UserCart = MutableMap<String, List<CartItem>>

// 이제 훨씬 더 읽기 쉬워졌습니다.
val userCart: UserCart = mutableMapOf()
```

`UserCart`라는 별칭 덕분에, 이 변수가 무엇을 담는 맵인지 그 의도가 훨씬 명확해졌습니다.

##### 함수 타입에 별칭 붙이기

함수 타입에 의미 있는 이름을 붙이면, 마치 인터페이스를 사용하는 것처럼 코드의 가독성을 높일 수 있습니다. 특히 콜백(Callback) 함수를 정의할 때 매우 유용합니다.

```kotlin
typealias OnClickListener = (View, MotionEvent) -> Boolean

// 훨씬 더 의미가 명확해진 함수 시그니처
fun setOnClickListener(listener: OnClickListener) { /* ... */ }

// 람다를 변수에 담을 때도 유용
val myListener: OnClickListener = { view, event ->
    // ... 클릭 처리 로직 ...
    true
}
setOnClickListener(myListener)
```

`OnClickListener`라는 이름만 보고도 이 파라미터가 어떤 역할을 하는지 즉시 이해할 수 있습니다.

#### `typealias`의 한계

`typealias`는 오직 이름만 바꾸는 것이므로, `typealias UserId = String` 이라고 선언해도 `UserId`와 `String`은 컴파일러에게 완전히 똑같은 타입으로 취급됩니다. 즉, `UserId` 타입의 변수에 일반 `String`을 할당해도 아무런 오류가 발생하지 않아 타입 안전성을 제공해주지는 못합니다. (타입 안전성까지 원한다면 `value class`와 같은 더 강력한 기능을 사용해야 합니다.)

타입 별칭은 코드의 가독성이 복잡한 타입 선언으로 인해 저하될 때, 이를 해결해 주는 간단하면서도 효과적인 도구입니다. 복잡한 제네릭이나 함수 타입을 다룰 때 적극적으로 활용하여 동료 개발자와 미래의 나를 위한 배려를 보여주시기 바랍니다.

-----

이것으로 코틀린의 특별한 클래스와 객체를 다룬 11장, 그리고 고급 문법과 함수형 프로그래밍을 탐험한 Part 3의 여정을 모두 마칩니다.

지금까지 당신은 객체 지향의 심화 개념부터 함수를 값처럼 다루는 함수형 프로그래밍의 정수, 그리고 상속의 대안이 되는 위임 패턴과 코틀린만의 독특한 언어 기능들까지 모두 섭렵했습니다. 이제 당신은 단순히 코틀린의 문법을 아는 것을 넘어, 다양한 패러다임을 조합하여 우아하고 효율적인 코드를 설계할 수 있는 깊이 있는 시야를 갖추게 되었습니다.

다음 Part 4에서는 현대 프로그래밍의 가장 큰 화두 중 하나인 동시성(Concurrency)을 코틀린이 얼마나 혁신적인 방식으로 해결하는지, '코루틴'의 세계로 함께 떠나보겠습니다.