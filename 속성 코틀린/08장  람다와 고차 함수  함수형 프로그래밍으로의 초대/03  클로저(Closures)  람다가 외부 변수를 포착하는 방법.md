### 클로저(Closures): 람다가 외부 변수를 포착하는 방법

고차 함수를 배우면서 우리는 `getMultiplier` 함수가 자기 자신만의 `factor` 값을 '기억'하는 새로운 함수를 반환하는 것을 보았습니다. `getMultiplier` 함수의 실행은 이미 끝났고, 그 함수의 파라미터였던 `factor`는 사라졌어야 하는데, 어떻게 반환된 람다는 `factor`의 값을 계속 알고 있는 것일까요?

이 마법의 비밀이 바로 \*\*클로저(Closure)\*\*입니다.

> **클로저의 정의**
> 자기 자신이 정의된 환경(스코프)으로부터 변수를 \*\*포획(capture)\*\*하여 자신의 코드 블록 안에서 계속 접근할 수 있는 함수. 코틀린에서는 람다나 지역 함수가 클로저의 역할을 수행합니다.

비유하자면, 람다는 자신이 태어난 환경의 "스냅샷" 📸을 찍어서 작은 가방에 담아 다니는 것과 같습니다. 설령 원래의 환경이 사라지더라도, 람다는 자신의 가방 속에 담아온 변수들을 계속해서 참조하고 사용할 수 있습니다.

-----

#### 클로저의 동작 원리

`getMultiplier` 예제를 다시 한번 자세히 들여다봅시다.

```kotlin
fun getMultiplier(factor: Int): (Int) -> Int {
    // 1. 람다가 정의되는 순간, 자신의 본문({ number -> number * factor })을 살펴봅니다.
    // 2. 본문이 외부 스코프에 있는 'factor' 변수를 사용한다는 것을 인지합니다.
    // 3. 람다는 이 'factor' 변수를 자신의 가방(클로저) 속에 '포획'합니다.
    return { number -> number * factor }
}

fun main() {
    // getMultiplier(2)가 호출되면, factor=2를 포획한 람다가 생성되어 double 변수에 할당됩니다.
    val double = getMultiplier(2)
    
    // getMultiplier(3)가 호출되면, factor=3을 포획한 또 다른 람다가 생성되어 triple 변수에 할당됩니다.
    val triple = getMultiplier(3)

    // getMultiplier 함수는 이미 실행이 끝났지만,
    // double 람다는 자신이 포획한 factor=2를 여전히 기억하고 있습니다.
    println(double(10)) // 10 * 2 = 20

    // triple 람다 역시 자신이 포획한 factor=3을 기억하고 있습니다.
    println(triple(10)) // 10 * 3 = 30
}
```

클로저는 람다가 단순한 코드 조각을 넘어, 자신이 생성된 \*\*문맥(Context)\*\*을 기억하는 독립적인 존재가 될 수 있도록 만들어 줍니다.

-----

#### 포획한 변수 수정하기

클로저는 외부 변수의 값을 읽는 것뿐만 아니라, **수정**하는 것도 가능합니다.

호출할 때마다 숫자가 1씩 증가하는 카운터 함수를 만들어 봅시다.

```kotlin
fun makeCounter(): () -> Int {
    // 'count'는 makeCounter 함수의 지역 변수입니다.
    var count = 0
    
    // 반환되는 람다는 이 'count' 변수를 포획합니다.
    return {
        count++ // 람다 내부에서 외부 변수 count를 수정합니다.
        count   // 수정된 값을 반환합니다.
    }
}

fun main() {
    val counter1 = makeCounter()
    println(counter1()) // 출력: 1
    println(counter1()) // 출력: 2
    println(counter1()) // 출력: 3

    // counter2는 counter1과 완전히 독립적인 자신만의 count 변수를 포획합니다.
    val counter2 = makeCounter()
    println(counter2()) // 출력: 1
    println(counter2()) // 출력: 2
    
    println(counter1()) // 출력: 4 (counter1의 count는 계속 유지됩니다.)
}
```

`makeCounter()` 함수를 호출할 때마다 새로운 `count` 변수와 그 변수를 포획한 새로운 람다가 생성됩니다. 각 클로저는 자신만의 독립적인 상태(`count`)를 유지하며 동작합니다. 이는 객체 없이도 상태를 캡슐화하는 매우 강력한 함수형 프로그래밍 패턴입니다.

사실 코틀린에서는 람다를 사용할 때마다 우리도 모르는 사이에 자연스럽게 클로저를 사용하고 있었던 것입니다. 클로저는 람다와 고차 함수가 함께 동작하는 방식을 이해하는 마지막 퍼즐 조각이며, 함수형 프로그래밍의 표현력을 한 단계 끌어올리는 핵심적인 개념입니다.