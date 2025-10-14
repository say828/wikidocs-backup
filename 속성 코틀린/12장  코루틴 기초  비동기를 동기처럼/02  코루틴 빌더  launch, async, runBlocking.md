### 코루틴 빌더: launch, async, runBlocking

`suspend` 함수가 코루틴의 세계에서만 호출될 수 있다면, 일반 함수가 지배하는 현실 세계에서 코루틴의 세계로 들어가는 최초의 '문'은 어떻게 열 수 있을까요? 이 역할을 하는 것이 바로 \*\*코루틴 빌더(Coroutine Builder)\*\*입니다.

코루틴 빌더는 새로운 코루틴을 시작하고, 일반 함수와 일시 중단 함수 사이의 다리를 놓아주는 함수입니다. 가장 기본적이고 중요한 세 가지 코루틴 빌더를 알아봅시다.

-----

#### 1\. `launch`: 결과를 기다리지 않는 작업

\*\*`launch`\*\*는 결과를 반환하지 않는 코루틴을 시작할 때 사용합니다. "실행하고 잊어버리는(fire and forget)" 방식의 작업을 위한 빌더입니다. 백그라운드에서 무언가를 업데이트하거나, 로그를 보내는 등 독립적으로 실행되면 충분한 작업에 적합합니다.

📲 **비유:** `launch`는 메시지를 전송하는 것과 같습니다. '전송' 버튼을 누르면 메시지가 백그라운드에서 날아가고, 우리는 답장을 기다리지 않고 즉시 다른 일을 할 수 있습니다.

```kotlin
import kotlinx.coroutines.*

fun main() {
    // GlobalScope는 애플리케이션 전체 생명주기를 가지는 코루틴 스코프입니다. (연습용으로만 사용 권장)
    GlobalScope.launch { 
        println("코루틴 시작!")
        delay(1000L) // 1초간 코루틴 일시 중단
        println("World!") 
    }
    
    println("Hello,") // launch 블록 밖의 코드는 바로 실행됨
    Thread.sleep(2000L) // 메인 스레드가 2초간 대기 (코루틴이 끝날 시간을 벌어줌)
    println("메인 스레드 종료")
}
```

**실행 결과:**

```
Hello,
코루틴 시작!
(1초 후)
World!
(1초 후)
메인 스레드 종료
```

`launch`가 호출되자마자 코루틴은 백그라운드에서 자신의 일을 시작하고, `main` 함수의 흐름은 멈추지 않고 계속 진행됩니다. `launch`는 시작된 코루틴을 제어할 수 있는 `Job` 객체를 반환합니다.

-----

#### 2\. `async`: 결과를 반환하는 작업

\*\*`async`\*\*는 `launch`와 비슷하게 코루틴을 시작하지만, 코루틴의 실행 결과(값)를 나중에 받아야 할 때 사용합니다. `async`는 `Deferred<T>`라는 특별한 객체를 즉시 반환하는데, 이는 '미래의 언젠가 T 타입의 결과값이 담길 약속 증서'와 같습니다.

📦 **비유:** `async`는 온라인으로 상품을 주문하는 것과 같습니다. 주문을 하면 즉시 '송장 번호(`Deferred`)'를 받고, 배송이 오는 동안(`코루틴 실행 중`) 다른 일을 할 수 있습니다. 상품이 꼭 필요해지면, 송장 번호를 가지고 '배송 완료(`await()`)'되기를 기다려 상품을 수령합니다.

```kotlin
import kotlinx.coroutines.*

suspend fun fetchUser(id: String): String {
    delay(1000L)
    return "User($id)"
}

fun main() = runBlocking { // 예제를 위해 runBlocking 사용
    val deferredUser: Deferred<String> = GlobalScope.async {
        fetchUser("user123")
    }

    println("사용자 정보를 기다리는 동안 다른 작업을 합니다...")
    
    // deferredUser의 결과가 준비될 때까지 코루틴을 일시 중단하고 기다림
    val user: String = deferredUser.await()
    
    println(user)
}
```

**실행 결과:**

```
사용자 정보를 기다리는 동안 다른 작업을 합니다...
(1초 후)
User(user123)
```

`async`는 비동기적으로 두 개 이상의 작업을 동시에 수행하고, 나중에 그 결과들을 조합해야 할 때 매우 강력한 힘을 발휘합니다. `await()`는 결과가 준비될 때까지 기다리는 `suspend` 함수입니다.

-----

#### 3\. `runBlocking`: 세상을 멈추는 다리

\*\*`runBlocking`\*\*은 이름 그대로 \*\*현재 스레드를 차단(block)\*\*하면서 새로운 코루틴을 시작하는 특별한 빌더입니다. `runBlocking` 내부의 코루틴과 그 자식 코루틴들이 모두 완료될 때까지, `runBlocking`을 호출한 스레드는 다른 일을 하지 못하고 그 자리에서 기다립니다.

🛋️ **비유:** `runBlocking`은 '대기실'과 같습니다. `main`이라는 외부인이 대기실에 들어오면, 대기실 안의 모든 용무(코루틴 작업)가 끝날 때까지 밖으로 나갈 수 없습니다.

**주요 용도:**

  * `main` 함수처럼, 동기적인 코드의 최상위 레벨에서 코루틴을 시작하고 그 결과를 기다려야 할 때.
  * `suspend` 함수를 테스트하는 단위 테스트(Unit Test) 코드에서.

**주의:** 스레드를 차단하므로, 안드로이드의 UI 스레드와 같이 절대 멈춰서는 안 되는 곳에서는 **절대 사용해서는 안 됩니다.**

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // main 함수를 runBlocking으로 감싸면...
    launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    
    // 더 이상 Thread.sleep()이 필요 없습니다!
    // runBlocking이 내부의 launch 코루틴이 끝날 때까지 main 스레드를 기다려줍니다.
}
```

**실행 결과:**

```
Hello,
(1초 후)
World!
```

이 세 가지 빌더는 코루틴의 세계로 들어가는 각기 다른 목적을 가진 문입니다. `launch`로 작업을 던져놓고, `async`로 결과물을 약속받으며, `runBlocking`으로 두 세계를 연결하는 방법을 이해하는 것이 코루틴의 첫걸음입니다.