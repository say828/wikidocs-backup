## 02\. 시간 의존적인 로직 테스트: TestCoroutineScheduler를 이용한 시간 제어

`runTest`의 자동 시간 진행 기능은 대부분의 비동기 테스트를 마법처럼 간단하게 만들어준다. 하지만 때로는 우리가 직접 시간의 흐름을 통제하며 "1초 후에는 이 상태여야 하고, 다시 500ms가 흐르면 저 상태가 되어야 한다"와 같이 시간의 특정 지점에서 발생하는 이벤트를 정밀하게 검증해야 하는 경우가 있다. 예를 들면 다음과 같은 로직이다.

  * **타임아웃(Timeout):** "특정 작업이 5초 안에 끝나지 않으면 타임아웃 예외를 던져야 한다."
  * **디바운싱(Debouncing):** "사용자가 키보드를 입력할 때, 마지막 입력 후 300ms 동안 추가 입력이 없으면 자동 검색 API를 호출해야 한다."
  * **폴링(Polling):** "백그라운드 작업의 상태를 1초마다 주기적으로 확인해야 한다."

이러한 시간 의존적인 로직을 `Thread.sleep` 없이 테스트하기 위해, 우리는 `runTest`의 마법 뒤에 숨어있는 \*\*`TestCoroutineScheduler`\*\*를 직접 제어하는 방법을 배워야 한다.

### **시간의 마술사: `TestCoroutineScheduler`**

`TestCoroutineScheduler`는 `runTest`가 사용하는 가상 시간의 흐름을 관리하는 실제 주체다. `runTest` 블록 내에서는 `this.testScheduler` 속성을 통해 이 스케줄러에 직접 접근할 수 있다. 스케줄러는 다음과 같은 핵심 메소드를 제공하여 시간을 수동으로 제어할 수 있게 해준다.

  * **`advanceTimeBy(delayTimeMillis: Long)`:** 가상 시간을 지정된 시간만큼 **앞으로 진행**시킨다. 이 시간 동안 실행되어야 할 모든 코루틴 작업이 즉시 실행된다.
  * **`runCurrent()`:** 현재 가상 시간에 대기 중인 모든 작업을 즉시 실행한다. 시간을 앞으로 진행시키지는 않는다.
  * **`currentTime`:** 현재 가상 시간을 밀리초 단위로 반환한다.

### **TDD 예시: 검색어 디바운싱 로직 테스트**

사용자의 빠른 타이핑에 의해 API가 과도하게 호출되는 것을 막기 위해, 마지막 입력 후 500ms가 지나야만 검색을 실행하는 `SearchService`를 TDD로 개발해보자.

**1. 실패하는 테스트로 시간의 흐름을 명세하라 (RED)**

먼저, "입력 후 500ms 안에 다른 입력이 들어오면 이전 검색은 취소되어야 한다"는 핵심 동작을 테스트로 작성한다.

```kotlin
class SearchServiceTest : BehaviorSpec({
    // runTest의 dispatcher를 사용하지 않고, 수동 제어를 위해 직접 생성한다.
    val testScheduler = TestCoroutineScheduler()
    val testDispatcher = StandardTestDispatcher(testScheduler)
    
    // 테스트 대상에 TestDispatcher를 주입할 수 있도록 CoroutineScope를 생성
    val testScope = TestScope(testDispatcher)

    // SUT
    lateinit var searchService: SearchService
    // Collaborator
    lateinit var searchApiClient: SearchApiClient

    beforeTest {
        searchApiClient = mockk(relaxed = true)
        searchService = SearchService(searchApiClient, testScope)
    }

    given("사용자가 매우 빠르게 여러 키워드를 입력했을 때") {
        `when`("마지막 입력 후 500ms가 지나기 전에 계속 입력하면") {
            searchService.search("T")
            testScheduler.advanceTimeBy(200) // 200ms 경과
            searchService.search("TD")
            testScheduler.advanceTimeBy(200) // 총 400ms 경과
            searchService.search("TDD")

            then("API 클라이언트는 전혀 호출되지 않아야 한다") {
                // 현재 시간에 대기중인 작업을 모두 실행
                testScheduler.runCurrent()
                verify(exactly = 0) { searchApiClient.search(any()) }
            }
        }

        `when`("마지막 입력 후 500ms가 지나면") {
            searchService.search("TDD")
            testScheduler.advanceTimeBy(501) // 501ms 경과

            then("오직 마지막 키워드로만 API 클라이언트가 한 번 호출되어야 한다") {
                verify(exactly = 1) { searchApiClient.search("TDD") }
            }
        }
    }
})
```

`runTest` 대신 `TestScope(StandardTestDispatcher(TestCoroutineScheduler()))`를 직접 만들어 사용함으로써, 우리는 시간의 흐름을 완전히 통제할 수 있게 되었다.

**2. 테스트를 통과시키는 코드 작성 (GREEN)**

이제 이 정교한 시간 기반 테스트를 통과시키는 `SearchService`를 구현한다. 코루틴의 `launch`, `Job.cancel`, `delay`를 조합하면 디바운싱 로직을 쉽게 구현할 수 있다.

```kotlin
class SearchService(
    private val searchApiClient: SearchApiClient,
    private val scope: CoroutineScope // 외부에서 CoroutineScope를 주입받음
) {
    private var searchJob: Job? = null

    fun search(keyword: String) {
        // 이전 검색 작업이 있다면 취소한다.
        searchJob?.cancel()
        
        // 새로운 검색 작업을 CoroutineScope 내에서 시작한다.
        searchJob = scope.launch {
            // 500ms를 기다린다. 이 시간 안에 cancel()이 호출되면 이 코루틴은 종료된다.
            delay(500)
            
            // 500ms 동안 취소되지 않았다면, 실제 API를 호출한다.
            searchApiClient.search(keyword)
        }
    }
}
```

`TestScope`를 주입받아 사용함으로써, 프로덕션 코드에서는 실제 `CoroutineScope`를, 테스트 코드에서는 `TestDispatcher`와 연결된 `TestScope`를 사용하여 코드를 변경 없이 테스트할 수 있다. 이제 모든 테스트가 성공적으로 통과한다.

-----

`TestCoroutineScheduler`를 직접 제어하는 것은 코루틴 테스트의 가장 고급 기술 중 하나다. 이 기술을 통해 우리는 시간이라는 까다로운 요소를 우리 테스트의 완벽한 통제하에 둘 수 있다. `Thread.sleep`에 의존하는 느리고 불안정한 테스트와 작별하고, 시간의 흐름을 밀리초 단위로 조종하며 복잡한 비동기 로직의 모든 시나리오를 빠르고 정확하게 검증하는 것, 이것이 바로 비동기 TDD 전문가의 방식이다.