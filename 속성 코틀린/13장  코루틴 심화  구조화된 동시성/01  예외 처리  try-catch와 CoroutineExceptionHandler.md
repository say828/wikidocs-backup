### 예외 처리: try-catch와 CoroutineExceptionHandler

아무리 잘 만든 코드라도 네트워크 오류, 잘못된 데이터 등 예기치 못한 문제로 인해 예외(Exception)가 발생할 수 있습니다. 동시성 환경에서 예외를 제대로 처리하지 않으면, 예외가 조용히 무시되어 원인 모를 버그를 만들거나 최악의 경우 애플리케이션 전체를 중단시킬 수 있습니다.

코틀린의 구조화된 동시성은 이러한 예외가 절대 길을 잃지 않고, 예측 가능한 방식으로 처리되도록 보장하는 강력한 메커니즘을 제공합니다.

-----

#### 코루틴의 기본 예외 전파 방식

구조화된 동시성 환경에서 예외는 다음과 같은 규칙에 따라 전파됩니다.

**"자식 코루틴에서 처리되지 않은 예외는 부모에게 전파된다."**

예외가 부모에게 전파되면, 부모는 다음과 같이 행동합니다.

1.  자신을 포함한 **모든 자식 코루틴들을 즉시 취소**시킵니다.
2.  자신 또한 실패 상태가 되고, 자신의 부모에게 예외를 다시 전파합니다.

이러한 '전체 실패(fail-fast)' 정책은 매우 중요합니다. 여러 코루틴이 협력하여 하나의 작업을 수행할 때, 그중 하나라도 실패하면 나머지 작업들도 더 이상 의미가 없거나 잘못된 상태로 계속 진행될 위험이 있기 때문입니다. 시스템 전체를 일관된 상태로 유지하기 위해 관련된 모든 작업을 함께 중단시키는 것입니다.

```kotlin
fun main() = runBlocking {
    val job = launch { // 부모 코루틴 (job)
        launch { // 자식 1
            println("자식 1: 시작")
            delay(1000L)
            println("자식 1: 종료") // 이 코드는 실행되지 않습니다.
        }
        launch { // 자식 2
            println("자식 2: 시작")
            delay(500L)
            throw IllegalStateException("자식 2에서 에러 발생!")
        }
    }
    job.join()
    println("모든 작업 완료")
}
```

**실행 결과:**
'자식 2'에서 예외가 발생하자마자, 이 예외는 부모인 `job`에게 전파됩니다. `job`은 즉시 자신의 다른 자식인 '자식 1'을 취소시키고, 자신도 실패하며 예외를 던집니다. 따라서 "자식 1: 종료" 메시지는 출력되지 않습니다.

-----

#### 예외 처리 전략

예외를 처리하는 방법은 크게 두 가지로 나뉩니다.

##### 1\. `try-catch`: 지역적으로 예외를 잡을 때

코루틴 내부에서 발생할 수 있는 '예상된' 예외를 처리하고, 프로그램의 흐름을 계속 이어나가고 싶을 때 사용합니다. 이는 일반적인 동기 코드의 `try-catch`와 완전히 동일하게 동작합니다.

**`launch`에서의 사용:**

```kotlin
scope.launch {
    try {
        println("위험한 작업을 시도합니다...")
        throw IOException("네트워크 연결 실패")
    } catch (e: IOException) {
        // 예외를 잡아서 처리했으므로, 부모에게 전파되지 않습니다.
        println("예외 발생: ${e.message}. 복구 로직을 수행합니다.")
    }
}
```

**`async`에서의 사용:**
`async`의 경우, 예외는 즉시 발생하지 않고 `Deferred` 객체 내부에 저장됩니다. 그리고 `await()`를 호출하는 시점에 저장되어 있던 예외가 던져집니다. 따라서 `try-catch`는 `await()` 호출을 감싸야 합니다.

```kotlin
val deferred = scope.async {
    // ...
    throw IndexOutOfBoundsException("데이터가 존재하지 않습니다.")
}

try {
    val result = deferred.await()
    println("결과: $result")
} catch (e: IndexOutOfBoundsException) {
    println("async 작업 실패: ${e.message}")
}
```

##### 2\. `CoroutineExceptionHandler`: 전역적으로 예외를 잡을 때

`try-catch`로 잡히지 않은, 즉 '처리되지 않은' 모든 예외가 스코프의 최상단까지 전파되었을 때 이를 처리하기 위한 최후의 보루입니다. `CoroutineContext`의 한 요소로 등록하여 사용합니다.

**언제 사용할까?**

  * **글로벌 로깅:** 모든 예상치 못한 에러를 로그 파일이나 크래시 리포팅 서비스에 기록할 때.
  * **최후의 복구:** 사용자에게 "알 수 없는 오류가 발생했습니다"와 같은 공통적인 오류 메시지를 보여줄 때.

<!-- end list -->

```kotlin
// 1. 핸들러 정의
val handler = CoroutineExceptionHandler { coroutineContext, exception ->
    // 이 람다는 처리되지 않은 예외가 발생했을 때 호출됩니다.
    println("처리되지 않은 예외 발생! context: $coroutineContext, exception: $exception")
}

// 2. 스코프 생성 시 컨텍스트에 핸들러 추가
val scope = CoroutineScope(Dispatchers.Default + handler)

// 3. 예외를 발생시키는 코루틴 실행
scope.launch {
    throw ArithmeticException("0으로 나눌 수 없습니다!")
}
```

**중요:** `CoroutineExceptionHandler`는 예외 전파를 **막지 않습니다.** 예외는 여전히 부모를 취소시키고 전파됩니다. 핸들러는 이 모든 과정이 일어난 후에, 프로그램이 비정상적으로 종료되기 직전에 호출되어 예외를 기록하거나 처리할 마지막 기회를 제공하는 '안전망'입니다.

구조화된 동시성의 예측 가능한 예외 전파 방식과, `try-catch` 및 `CoroutineExceptionHandler`를 조합하면 어떤 상황에서도 예외를 놓치지 않는 견고한 동시성 애플리케이션을 만들 수 있습니다.