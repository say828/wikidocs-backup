### Job과 Deferred: 코루틴의 생명주기 관리

`launch`나 `async`와 같은 코루틴 빌더를 호출하면, 단순히 코드 블록이 실행되는 것을 넘어 코루틴의 상태와 생명주기를 나타내는 특별한 객체가 반환됩니다. 이 객체들을 통해 우리는 실행 중인 코루틴을 제어하고 상호작용할 수 있습니다.

-----

#### Job: 코루틴의 핸들(Handle)

\*\*`Job`\*\*은 `launch` 빌더가 반환하는 객체로, **하나의 코루틴 그 자체를 대표하는 핸들(handle) 또는 식별자**입니다. `Job`을 통해 우리는 코루틴의 현재 상태를 확인하거나, 시작된 코루틴을 취소하는 등 생명주기를 관리할 수 있습니다.

`Job`은 다음과 같은 생명주기 상태를 가집니다.

  * **New**: 생성되었지만 아직 실행되지 않은 상태
  * **Active**: 실행 중인 상태
  * **Completing**: 내부의 모든 자식 코루틴들이 완료되기를 기다리는 중간 상태
  * **Completed**: 성공적으로 실행을 마친 상태
  * **Cancelling**: 취소 요청을 받고 취소 작업을 진행 중인 상태
  * **Cancelled**: 취소가 완료된 상태

`Job`의 가장 중요한 역할 중 하나는 **코루틴을 취소**하는 것입니다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job: Job = launch {
        try {
            repeat(1000) { i ->
                println("Job: I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            // 취소될 때 항상 실행되는 정리 코드
            println("Job: I'm running in finally!")
        }
    }

    delay(1300L) // 잠시 기다렸다가
    println("main: I'm tired of waiting!")
    job.cancel() // 코루틴을 취소합니다.
    job.join()   // 코루틴이 완전히 취소될 때까지 기다립니다.
    println("main: Now I can quit.")
}
```

**실행 결과:**

```
Job: I'm sleeping 0 ...
Job: I'm sleeping 1 ...
Job: I'm sleeping 2 ...
main: I'm tired of waiting!
Job: I'm running in finally!
main: Now I can quit.
```

`job.cancel()`이 호출되자, `launch` 블록 내에서 실행되던 코루틴은 `CancellationException`을 발생시키며 중단됩니다. `finally` 블록은 예외 발생 여부와 상관없이 항상 실행되므로, 여기서 자원을 정리하는 등의 마무리 작업을 할 수 있습니다. `job.join()`은 해당 `Job`이 완료(성공 또는 취소)될 때까지 현재 코루틴을 일시 중단시키는 `suspend` 함수입니다.

-----

#### Deferred: 결과를 담는 Job

**`Deferred`는 `async` 빌더가 반환하는 객체입니다. `Deferred`는 `Job`의 모든 기능을 가지면서(`Job`을 상속받습니다**), 추가적으로 코루틴의 실행 **결과(값)를 담을 수 있는 능력**을 가집니다. `Deferred<T>`는 언젠가 `T` 타입의 결과값을 돌려줄 것이라는 약속입니다.

`Deferred`의 가장 중요한 역할은 `await()` 함수를 통해 **결과를 가져오는 것**입니다.

```kotlin
import kotlinx.coroutines.*

suspend fun calculateResult(): Int {
    delay(1000L)
    return 42
}

fun main() = runBlocking {
    val deferred: Deferred<Int> = async {
        calculateResult()
    }

    println("계산이 진행되는 동안 다른 작업을 할 수 있습니다.")

    // await()를 호출하여 결과가 준비될 때까지 기다립니다.
    val result = deferred.await()
    println("계산 결과: $result")

    // Deferred는 Job이므로, 취소도 가능합니다.
    // deferred.cancel()
}
```

`await()`는 `suspend` 함수이므로, 결과가 준비될 때까지 현재 코루틴을 차단하지 않고 일시 중단시킵니다.

-----

#### Job vs. Deferred 요약

| 구분 | `Job` | `Deferred<T>` |
| :--- | :--- | :--- |
| **반환하는 빌더**| `launch` | `async` |
| **주요 목적** | 코루틴의 생명주기 제어 (취소, 상태 확인 등) | `Job`의 모든 기능 + **결과값 반환** |
| **결과 가져오기**| 불가능 | `await()` 메서드를 통해 가능 |
| **상속 관계** | - | `Job`을 상속함 |

`Job`과 `Deferred`는 실행 중인 비동기 작업을 추적하고 제어할 수 있는 필수적인 도구입니다. 이를 통해 우리는 더 안정적이고 예측 가능한 동시성 코드를 작성할 수 있습니다.