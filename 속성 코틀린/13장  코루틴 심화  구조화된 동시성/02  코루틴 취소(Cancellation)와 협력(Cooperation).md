### 코루틴 취소(Cancellation)와 협력(Cooperation)

`job.cancel()`을 호출하는 것은 코루틴을 강제로 즉시 중단시키는 전원 차단 스위치가 아닙니다. 코루틴의 취소는 \*\*'협력적(cooperative)'\*\*으로 동작합니다. 이는 `cancel()`이 코루틴에게 "이제 작업을 중단해달라"는 **요청**을 보내는 것과 같으며, 요청을 받은 코루틴이 이를 인지하고 스스로 작업을 멈춰야 함을 의미합니다.

이러한 협력적 모델은 매우 중요합니다. 코루틴이 갑자기 중단되면 사용하던 파일이나 네트워크 연결과 같은 리소스를 제대로 닫지 못해 시스템을 불안정한 상태로 만들 수 있기 때문입니다. 협력적 취소는 코루틴에게 작업을 마무리하고 뒷정리(cleanup)를 할 기회를 줍니다.

-----

#### 취소에 협력하는 방법

실행 중인 코루틴이 취소 요청에 응답하도록 만드는 방법은 크게 두 가지입니다.

##### 1\. `kotlinx.coroutines`의 일시 중단 함수 사용하기

가장 쉽고 일반적인 방법입니다. `delay()`, `yield()`, `withContext()` 등 `kotlinx.coroutines` 라이브러리에서 제공하는 대부분의 일시 중단 함수들은 \*\*취소를 지원(cancellable)\*\*합니다. 즉, 이 함수들은 내부적으로 코루틴의 현재 상태를 확인하여, 만약 취소 요청이 들어왔다면 `CancellationException`이라는 특별한 예외를 던져 코루틴의 실행을 중단시킵니다.

따라서 긴 작업을 수행하는 코루틴 내부에 `delay()`와 같은 함수를 주기적으로 호출하는 것만으로도, 여러분의 코루틴은 취소에 자동으로 협력하게 됩니다.

```kotlin
val job = launch {
    repeat(100) { i ->
        println("반복 작업 중... $i")
        delay(500L) // 이 함수가 취소 요청을 확인하는 '협력 지점'이 됩니다.
    }
}

delay(1200L) // 1.2초 후
job.cancel() // 취소 요청
```

위 코드에서 `job.cancel()`이 호출되면, `repeat` 루프 안에서 다음 `delay(500L)`가 실행되는 시점에 `CancellationException`이 발생하여 루프가 중단됩니다.

##### 2\. 명시적으로 상태 확인하기

만약 코루틴이 CPU 자원만 집중적으로 사용하는 복잡한 연산을 수행하느라 일시 중단 함수를 전혀 호출하지 않는다면 어떨까요? 이런 코루틴은 취소 요청을 확인할 기회가 없으므로, `cancel()`을 호출해도 작업을 멈추지 않고 계속 실행하게 됩니다.

이 경우, 개발자는 직접 코루틴의 상태를 확인하여 취소에 협력해야 합니다.

  * **`isActive` 프로퍼티:** `CoroutineScope` 내에서 접근할 수 있는 `isActive`는 코루틴이 현재 활성 상태인지를 나타내는 `Boolean` 값입니다. 취소되면 `false`가 됩니다. 반복문의 조건으로 사용하여 주기적으로 확인하는 것이 좋습니다.
  * **`ensureActive()` 함수:** `isActive`가 `false`이면 즉시 `CancellationException`을 던지는 함수입니다. 코드 중간중간에 검문소를 설치하는 것처럼 사용할 수 있습니다.

**협력하지 않는 코드의 문제점:**

```kotlin
// 이 코드는 delay()가 없어서 취소되지 않고 5초 동안 계속 실행됩니다.
val job = launch(Dispatchers.Default) {
    var i = 0
    while (i < 5) {
        // CPU만 사용하는 복잡한 작업 (Thread.sleep은 코루틴 취소와 무관)
        Thread.sleep(1000L) 
        i++
        println("작업 진행 중... $i")
    }
}
delay(2500L)
job.cancel() // 2.5초 뒤에 취소 요청을 보내지만...
job.join()
println("작업 종료")
```

**협력하도록 수정한 코드:**

```kotlin
val job = launch(Dispatchers.Default) {
    var i = 0
    // while문의 조건으로 isActive를 직접 확인합니다.
    while (i < 5 && isActive) { 
        Thread.sleep(1000L)
        i++
        println("작업 진행 중... $i")
    }
}
delay(2500L)
job.cancel()
job.join()
println("작업 종료")
```

이제 `job.cancel()`이 호출되면 `isActive`가 `false`가 되어 `while` 루프가 즉시 종료됩니다.

-----

#### 취소 시 리소스 정리하기: `finally` 블록

코루틴이 `CancellationException`에 의해 종료될 때, 열었던 파일이나 네트워크 연결을 닫는 등의 정리 작업은 어떻게 보장할 수 있을까요? 정답은 `try...finally` 블록입니다.

`finally` 블록은 `try` 블록이 정상적으로 끝나든, 예외로 인해 중단되든, **취소로 인해 중단되든 관계없이 항상 실행됨을 보장**합니다.

```kotlin
val job = launch {
    try {
        println("리소스 획득!")
        repeat(1000) { i ->
            println("작업 중... $i")
            delay(500L)
        }
    } finally {
        // 이 블록은 코루틴이 취소될 때 반드시 실행됩니다.
        println("리소스 정리 완료!")
    }
}

delay(1300L)
job.cancelAndJoin() // cancel()과 join()을 한번에 호출
println("메인: 코루틴 취소 완료")
```

**참고:** `finally` 블록 안에서는 코루틴이 이미 취소 상태이므로 일반적인 일시 중단 함수를 호출할 수 없습니다. 만약 정리 작업에 일시 중단 함수가 꼭 필요하다면, `withContext(NonCancellable) { ... }`으로 코드를 감싸야 합니다.

결론적으로, 잘 작성된 코루틴은 취소에 적극적으로 협력해야 하며, `try-finally`를 통해 어떤 상황에서도 리소스를 안전하게 해제해야 합니다. 이것이 안정적인 동시성 프로그램을 만드는 핵심 원칙입니다.