### noinline과 crossinline의 사용 시점

`inline`은 람다 사용의 성능 오버헤드를 없애주는 강력한 기능이지만, 때로는 `inline` 함수의 모든 람다 파라미터를 인라이닝하고 싶지 않거나, 인라이닝으로 인해 발생하는 특정 동작을 제어해야 할 필요가 있습니다. 이때 사용하는 것이 바로 `noinline`과 `crossinline` 변경자입니다.

-----

#### noinline: 인라이닝을 원치 않는 람다

`inline`으로 선언된 함수의 파라미터로 전달된 람다는 기본적으로 모두 인라이닝됩니다. 하지만 특정 람다는 인라이닝하지 않고, 일반적인 함수처럼 **객체로 존재하게 만들어야** 할 때가 있습니다.

**`noinline`을 사용하는 경우:**

  * 인라인 함수의 파라미터로 받은 람다를, **다른 일반 함수에 인자로 전달**하거나 **변수에 저장**해야 할 때.

인라이닝된 람다는 바이트코드 상에서 객체로 존재하지 않고 코드로 펼쳐지기 때문에, 객체를 필요로 하는 변수나 다른 함수에 전달할 수 없습니다. 이때 `noinline`을 사용하면 해당 람다만 인라이닝에서 제외하여 이 문제를 해결할 수 있습니다.

```kotlin
// 이 함수는 inline 함수가 아니므로, 파라미터로 함수 '객체'를 받습니다.
fun logAndExecute(task: () -> Unit) {
    println("작업 로깅...")
    task()
}

// execute 함수는 inline이지만, logTask 파라미터는 noinline으로 지정
inline fun execute(
    mainTask: () -> Unit,
    noinline logTask: () -> Unit
) {
    println("메인 작업 시작")
    mainTask() // mainTask의 본문은 이곳에 복사됨 (인라인)
    
    // logTask는 noinline이므로, 객체로 취급되어 다른 함수에 전달 가능
    logAndExecute(logTask)
}

fun main() {
    execute(
        mainTask = { println("핵심 로직 수행") },
        logTask = { println("이 작업은 로그로 남겨야 해") }
    )
}
```

`mainTask`는 인라이닝되어 객체 생성 없이 코드가 복사되지만, `logTask`는 `noinline` 덕분에 일반 함수 객체로 생성되어 `logAndExecute` 함수에 성공적으로 전달될 수 있습니다.

-----

#### crossinline: 비-지역 반환(Non-local Return) 금지

인라인된 람다의 강력한 특징 중 하나는, 람다 내부에서 `return`을 사용하면 람다뿐만 아니라 그 람다를 호출한 바깥 함수까지 종료시키는 \*\*비-지역 반환(Non-local return)\*\*이 가능하다는 것입니다.

하지만 람다가 호출되는 시점이 불분명하다면 어떨까요? 예를 들어, 인라인된 람다가 즉시 실행되지 않고 다른 객체에 전달되어 나중에 실행될 수도 있습니다. 이런 상황에서 비-지역 반환이 일어나면 이미 종료된 함수를 빠져나가려는 로직상의 모순이 발생합니다.

\*\*`crossinline`\*\*은 이러한 위험을 방지하기 위한 안전장치입니다. `crossinline`으로 지정된 람다는 여전히 **인라이닝의 성능상 이점은 누리지만, 비-지역 반환(`return`)은 금지됩니다.**

**`crossinline`을 사용하는 경우:**

  * 인라인 함수의 람다 파라미터가, 함수 본문에서 **직접 호출되지 않고 다른 실행 컨텍스트(예: 다른 람다나 객체)를 통해** 호출될 때.

<!-- end list -->

```kotlin
inline fun runWithDelayedStart(crossinline block: () -> Unit) {
    println("3초 뒤에 작업을 시작합니다...")
    
    // block 람다가 즉시 실행되지 않고, Timer의 콜백(다른 실행 컨텍스트)에서 실행됨
    Timer().schedule(object : TimerTask() {
        override fun run() {
            block()
        }
    }, 3000L)
}

fun main() {
    runWithDelayedStart {
        println("작업 실행!")
        // return // 컴파일 오류! crossinline 람다에서는 비-지역 반환을 할 수 없습니다.
    }
    println("main 함수는 계속 진행됩니다...")
}
```

`block` 람다는 `runWithDelayedStart` 함수가 종료된 후 3초 뒤에 호출될 수 있습니다. 만약 여기서 비-지역 반환이 허용된다면 이미 사라진 `main` 함수를 반환하려는 시도가 되어 위험합니다. `crossinline`은 이러한 위험한 `return` 사용을 컴파일 시점에 미리 막아줍니다.

| 변경자 | 인라이닝 여부 | 객체로 전달 가능? | 비-지역 반환 가능? |
| :--- | :--- | :---: | :---: |
| (기본) | **O** | X | **O** |
| `noinline` | X | **O** | X |
| `crossinline`| **O** | **O** | X |

`noinline`과 `crossinline`은 `inline`이라는 강력한 기능을 더욱 세밀하고 안전하게 제어할 수 있도록 돕는 전문가용 도구입니다.

-----

이것으로 함수형 프로그래밍으로의 초대였던 8장을 마칩니다. 이제 당신은 람다와 고차 함수라는 강력한 무기를 손에 쥐었고, 그 내부 동작 원리와 제어 방법까지 이해하게 되었습니다. 다음 장에서는 이 무기들을 가지고 코틀린 표준 라이브러리라는 거대한 무기고를 탐험하며 실전 능력을 키워보겠습니다.