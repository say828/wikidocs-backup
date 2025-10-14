## 03\. 동시성 문제(Race Condition, Deadlock)를 재현하고 검증하는 테스트 케이스 설계

비동기 프로그래밍의 가장 어두운 그림자는 바로 **동시성(Concurrency)** 문제다. 여러 스레드나 코루틴이 공유된 자원(객체, 변수)에 동시에 접근하려 할 때 발생하는 \*\*경쟁 상태(Race Condition)\*\*는 시스템을 예측 불가능한 오류와 데이터 손상으로 이끄는 주범이다. 이러한 버그는 '가끔' 발생하고 재현하기가 극도로 어려워 '하이젠버그(Heisenbug)'라고도 불린다.

"재현이 어려운 버그를 TDD로 어떻게 다룰 수 있는가?" 이 질문에 대한 전문가의 대답은 명확하다. **우리는 버그를 재현하려 애쓰는 것이 아니라, 버그가 애초에 발생할 수 없는 설계를 TDD로 강제해야 한다.** 즉, 동시성 제어 메커니즘(Mutex, Atomic 변수 등)이 올바르게 적용되었음을 증명하는, **결정론적인(Deterministic)** 테스트를 작성하는 것이다.

### **Case Study: 경쟁 상태에 놓인 요청 카운터**

매우 간단한 요청 카운터 클래스를 생각해보자. 여러 코루틴이 동시에 `increment()`를 호출하는 상황이다.

**나쁜 예: 동시성에 취약한 코드**

```kotlin
class UnsafeRequestCounter {
    var count = 0
        private set

    // 이 메소드는 원자적(atomic)이지 않다!
    fun increment() {
        count++ // 1. 읽기(read) 2. 증가(modify) 3. 쓰기(write)의 3단계로 동작
    }
}
```

`count++` 연산은 내부적으로 값을 읽고, 1을 더하고, 다시 쓰는 세 단계로 이루어진다. 만약 두 코루틴이 동시에 `count`가 5일 때 이 메소드를 호출하면, 둘 다 5를 읽고, 둘 다 6을 만들어 쓴다. 최종 결과는 7이 아닌 6이 되는 끔찍한 데이터 유실이 발생한다.

### **경쟁 상태를 증명하는 (나쁜) 테스트**

이 문제를 증명하기 위해 수백 개의 코루틴을 동시에 실행하는 테스트를 작성할 수는 있다.

```kotlin
@Test
fun `이 테스트는 때때로 실패할 것이다`() = runBlocking {
    val counter = UnsafeRequestCounter()
    
    // Dispatchers.IO는 실제 멀티 스레드 환경에서 코루틴을 실행
    withContext(Dispatchers.IO) {
        repeat(1000) {
            launch {
                counter.increment()
            }
        }
    }
    
    // 이상적으로는 1000이 되어야 하지만, 경쟁 상태 때문에 그보다 작은 값이 나올 수 있다.
    println("Final count: ${counter.count}") 
}
```

이 테스트는 실행할 때마다 결과가 달라지며, 어떨 때는 성공하기도 한다. **FIRST** 원칙의 **Repeatable**을 위반하는 이런 테스트는 절대 작성해서는 안된다. 이것은 단지 문제가 존재함을 보여주기 위한 시연일 뿐이다.

### **TDD for 동시성 안전 설계 (Mutex)**

이제 이 문제를 해결하기 위해 코루틴이 제공하는 동기화 도구인 `Mutex`를 사용하여 `ThreadSafeRequestCounter`를 TDD로 개발해보자. 우리의 목표는 `Mutex`가 임계 영역(Critical Section)을 제대로 보호하는지를 **결정론적으로** 검증하는 것이다.

**1. 실패하는 테스트로 '락(Lock)'의 동작을 명세하라 (RED)**

`TestDispatcher`의 정교한 제어 능력을 활용하여, 여러 코루틴이 정확히 '동시에' 락을 획득하려 시도하는 상황을 시뮬레이션한다.

```kotlin
class ThreadSafeRequestCounterTest : FunSpec({
    test("여러 코루틴이 동시에 increment를 호출해도 count는 정확해야 한다") = runTest {
        val counter = ThreadSafeRequestCounter()
        val dispatcher = StandardTestDispatcher(testScheduler)
        
        // given: 100개의 코루틴이 동시에 실행되도록 준비
        val jobs = List(100) {
            launch(dispatcher) { // 제어 가능한 TestDispatcher에서 코루틴 실행
                counter.increment()
            }
        }

        // when: 대기 중인 모든 코루틴을 실행
        testScheduler.runCurrent()
        jobs.joinAll() // 모든 코루틴이 끝날 때까지 기다림

        // then: 최종 count는 정확히 100이어야 한다.
        counter.count shouldBe 100
    }
})
```

이 테스트는 `ThreadSafeRequestCounter`가 아직 존재하지 않거나, 내부의 `Mutex` 구현이 없으면 실패할 것이다.

**2. `Mutex`로 테스트를 통과시켜라 (GREEN)**

이제 `Mutex.withLock`을 사용하여 임계 영역을 보호하는 코드를 작성한다.

```kotlin
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

class ThreadSafeRequestCounter {
    private val mutex = Mutex()
    var count = 0
        private set

    suspend fun increment() {
        // withLock 블록은 한 번에 하나의 코루틴만 진입하도록 보장한다.
        mutex.withLock {
            count++
        }
    }
}
```

`TestDispatcher`는 코루틴들을 순차적으로 스케줄링하지만, `mutex.withLock`이 동시 접근을 막고 순서를 보장해주는 로직이 올바르게 구현되었음을 이 테스트는 증명한다. `runTest` 덕분에 이 복잡한 동시성 테스트조차도 매우 빠르게 실행된다.

-----

직접적으로 경쟁 상태나 교착 상태(Deadlock)를 재현하여 테스트하는 것은 거의 불가능하며 바람직하지도 않다. 대신, TDD는 우리에게 **동시성 문제를 회피하는 안전한 설계 패턴**을 적용하도록 유도하고, 그 패턴의 **정확성을 검증**하는 데 집중하게 한다. `kotlinx-coroutines-test`가 제공하는 정교한 시간 및 디스패처 제어 기능은 이러한 결정론적 동시성 테스트를 작성하는 데 필수적인 도구다.

이것으로 우리는 현대 아키텍처의 가장 깊고 어두운 바다인 비동기 세계의 탐험을 마쳤다. 우리는 이제 시간과 순서에 구애받지 않는 코드의 동작을 TDD를 통해 완벽하게 예측하고 통제할 수 있게 되었다. 다음 장에서는 개별 시스템의 내부 복잡성을 넘어, 여러 시스템이 함께 춤을 추는 **마이크로서비스 아키텍처**의 세계에서 TDD를 어떻게 적용할지 알아볼 것이다.