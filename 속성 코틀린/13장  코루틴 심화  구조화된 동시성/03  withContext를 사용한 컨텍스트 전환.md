### withContext를 사용한 컨텍스트 전환

우리는 앞서 `Dispatcher`를 통해 코루틴이 실행될 스레드를 지정하는 법을 배웠습니다. 그렇다면 코루틴이 실행되는 도중에 스레드를 바꿔야 할 때는 어떻게 해야 할까요? 예를 들어, UI 스레드에서 시작된 작업이 중간에 네트워크 통신을 위해 I/O 스레드로 갔다가, 그 결과를 다시 UI 스레드에 표시해야 하는 경우는 매우 흔합니다.

이러한 **실행 컨텍스트(스레드) 전환**을 위해 코틀린이 제공하는 가장 표준적이고 안전한 도구가 바로 **`withContext`** 함수입니다.

-----

#### `withContext`의 핵심 역할

`withContext`는 지정된 코루틴 컨텍스트(주로 디스패처)에서 코드 블록을 실행하고, 그 결과를 반환하는 일시 중단 함수입니다.

**`val result = withContext(Dispatcher) { ... }`**

이 함수는 다음과 같은 일을 보장합니다.

1.  **컨텍스트 전환:** 현재 실행 중인 코루틴을 `withContext`에 전달된 디스패처로 즉시 전환합니다.
2.  **블록 실행:** 주어진 람다 코드 블록을 새로 전환된 디스패처 위에서 실행합니다.
3.  **결과 반환:** 람다 블록의 마지막 표현식의 결과를 `withContext` 함수의 반환 값으로 돌려줍니다.
4.  **자동 복귀:** 람다 블록의 실행이 끝나면(성공하든, 예외가 발생하든), 코루틴은 **원래의 디스패처로 안전하게 복귀**합니다.

**비유:** `withContext`는 마치 조립 라인(`Main 스레드`)에서 일하던 작업자가 특수 공구가 필요한 작업을 위해 잠시 전문 작업대(`IO 스레드`)로 자리를 옮겨 일을 마친 뒤, 다시 원래 자신의 자리로 돌아와 다음 일을 이어가는 것과 같습니다.

```kotlin
suspend fun fetchUserData(): String {
    // 현재 스레드가 무엇이든 간에,
    // 이 블록 안에서는 Dispatchers.IO 스레드에서 실행됨을 보장받는다.
    val userName = withContext(Dispatchers.IO) {
        println("네트워크 요청 시작: ${Thread.currentThread().name}")
        delay(1000L) // 실제 네트워크 요청을 흉내
        "홍길동" // 이 값이 withContext의 반환 값이 됨
    }

    // withContext가 끝나면 원래 스레드로 돌아온다.
    println("원래 스레드로 복귀: ${Thread.currentThread().name}")
    return userName
}

fun main() = runBlocking {
    println("작업 시작: ${Thread.currentThread().name}")
    val user = fetchUserData()
    println("획득한 사용자: $user")
}
```

-----

#### 구조화된 동시성과의 관계

`withContext`는 새로운 코루틴을 만드는 것이 아니라, **현재 실행 중인 코루틴의 실행 컨텍스트만 잠시 바꾸는 것**입니다. 따라서 `withContext`는 부모 코루틴의 `Job`을 그대로 물려받으며, 구조화된 동시성 원칙을 완벽하게 따릅니다.

  * **취소 전파:** `withContext`를 호출한 부모 코루틴이 취소되면, `withContext` 내부에서 실행 중이던 작업 역시 즉시 취소됩니다.

<!-- end list -->

```kotlin
val job = launch {
    println("부모 코루틴 시작")
    withContext(Dispatchers.Default) {
        println("withContext 블록 시작")
        delay(2000L) // 이 작업은 완료되지 못함
        println("withContext 블록 종료")
    }
    println("부모 코루틴 종료")
}

delay(1000L)
job.cancelAndJoin() // 1초 후 부모 코루틴을 취소
println("취소 완료")
```

`job.cancel()`이 호출되자마자 `withContext` 블록 안에서 실행되던 `delay`가 취소되어, "withContext 블록 종료" 메시지는 출력되지 않습니다.

-----

#### `withContext` vs `async`

"결과를 반환하는 `withContext`는 `async { ... }.await()`와 무엇이 다른가?" 라는 의문이 들 수 있습니다. 둘은 비슷해 보이지만 목적이 다릅니다.

  * **`async`는 동시성/병렬성**을 위한 것입니다. 여러 작업을 **동시에** 실행하고 나중에 그 결과들을 조합할 때 사용합니다. `async`를 호출하면, 코드 블록이 백그라운드에서 실행되는 동안 현재 코루틴은 다른 작업을 계속할 수 있습니다.

    ```kotlin
    // 두 개의 네트워크 요청을 동시에 시작
    val userDeferred = async { apiClient.fetchUser() }
    val productsDeferred = async { apiClient.fetchProducts() }

    // 두 작업이 모두 끝날 때까지 기다렸다가 결과를 조합
    val user = userDeferred.await()
    val products = productsDeferred.await()
    ```

  * **`withContext`는 순차적인 코드의 스레드 전환**을 위한 것입니다. 현재 코루틴이 `withContext` 블록이 끝날 때까지 **기다렸다가(일시 중단)** 다음 코드를 실행합니다. 코드의 흐름은 여전히 순차적입니다.

    ```kotlin
    // 사용자 정보를 먼저 가져온 뒤,
    val user = withContext(Dispatchers.IO) { apiClient.fetchUser() }
    // 그 사용자 ID를 이용해 추천 상품 목록을 가져온다.
    val recommended = withContext(Dispatchers.IO) { apiClient.fetchRecommendations(user.id) }
    ```

**결론적으로,** 여러 작업을 병렬로 실행할 필요 없이, 단일 작업의 실행 스레드만 잠시 바꾸고 싶을 때는 `async`보다 `withContext`를 사용하는 것이 더 간단하고 명확한 선택입니다.