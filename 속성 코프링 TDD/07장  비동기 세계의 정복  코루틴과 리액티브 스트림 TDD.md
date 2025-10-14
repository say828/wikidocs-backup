# 07장: 비동기 세계의 정복: 코루틴과 리액티브 스트림 TDD

지금까지 우리가 TDD로 구축한 시스템은 대부분 '요청-응답' 모델 기반의 동기(Synchronous)적인 세상에 속해 있었다. 하지만 대규모 트래픽을 처리하고, 사용자에게 빠른 응답성을 제공하며, 시스템 자원을 효율적으로 사용해야 하는 현대 백엔드 애플리케이션의 세계는 점점 더 \*\*비동기(Asynchronous)\*\*적으로 변해가고 있다.

코틀린의 코루틴(Coroutine)은 이러한 비동기 프로그래밍의 복잡성을 혁신적으로 단순화시킨 강력한 도구다. 하지만 비동기 코드는 본질적으로 테스트하기 어렵다는 큰 난제를 안고 있다. 코드의 실행 순서와 시간이 보장되지 않아 테스트가 불안정해지기(flaky) 쉽고, `Thread.sleep()`과 같은 안티패턴에 의존하여 테스트가 느려지기 일쑤다.

이 장에서는 이러한 비동기 세계의 혼돈에 질서를 부여하는 방법을 배운다. 우리는 코루틴 테스트를 위한 공식 라이브러리인 `kotlinx-coroutines-test`를 깊이 있게 탐구하고, 시간의 흐름마저 통제하여 비동기 코드를 마치 동기 코드처럼 빠르고, 안정적이며, 예측 가능하게 테스트하는 기술을 완벽하게 마스터할 것이다.

-----

## 00\. 코루틴 테스트의 본질: runTest와 TestDispatcher의 동작 원리

코루틴 테스트의 세계를 정복하기 위해, 우리는 반드시 두 가지 핵심 무기의 동작 원리를 이해해야 한다. 바로 `runTest` 코루틴 빌더와 그 심장부에서 뛰고 있는 `TestDispatcher`다. 이 둘은 비동기 테스트의 가장 큰 적인 '불확실성'과 '느린 속도'를 제거하기 위해 탄생했다.

### **나쁜 예: `runBlocking`과 `Thread.sleep`의 함정**

코루틴을 처음 테스트하는 개발자는 `runBlocking`을 사용하여 테스트 스레드를 블로킹하고, `delay`와 같은 비동기 작업을 기다리기 위해 `Thread.sleep`을 사용하는 실수를 저지르곤 한다.

```kotlin
// 절대 이렇게 테스트하지 마세요!
@Test
fun `getProfile_BAD - Thread-sleep을 사용한 나쁜 테스트`() = runBlocking {
    val service = UserProfileService()
    
    // 비동기 작업 시작
    launch { service.getProfile("1") }
    
    Thread.sleep(1500) // 1초의 delay보다 길게, '감'에 의존해서 기다린다.
    
    // ... 검증 로직 ...
}
```

이 테스트는 **느리고(1.5초 소요), 불안정하며(네트워크가 더 느리면 실패), 신뢰할 수 없다.**

### **현대적인 해법: `runTest`와 가상 시간(Virtual Time)**

`kotlinx-coroutines-test` 라이브러리가 제공하는 `runTest`는 테스트를 위한 전용 코루틴 빌더다. 그 마법의 핵심은 \*\*가상 시간(Virtual Time)\*\*이라는 개념에 있다.

`runTest` 블록 내에서 코루틴은 실제 시계가 아닌, 테스트 라이브러리가 완벽하게 통제하는 가상의 시계를 따라 동작한다. `delay(1_000_000)` (백만 초)와 같은 코드가 있더라도, `runTest`는 즉시 가상 시간을 백만 초 후로 '점프'시켜 코드를 마저 실행한다. 덕분에 시간과 관련된 비동기 코드가 **거의 즉시** 완료된다.

**예시: `runTest`를 사용한 올바른 테스트**

```kotlin
// 프로덕션 코드
class UserProfileService {
    suspend fun getProfile(userId: String): UserProfile {
        delay(1000) // 네트워크 호출 등 1초가 걸리는 I/O 작업을 시뮬레이션
        return UserProfile(id = userId, name = "TDD Master")
    }
}

// 테스트 코드
class UserProfileServiceTest {
    @Test
    fun `프로필을 조회하면 사용자 정보가 반환된다`() = runTest {
        // given
        val service = UserProfileService()
        
        // when: suspend 함수 호출
        val profile = service.getProfile("1")
        
        // then
        profile.name shouldBe "TDD Master"
        
        // delay(1000)이 있었음에도, 이 테스트는 수 밀리초 안에 즉시 완료된다!
    }
}
```

`runTest`는 또한 테스트 블록 내에서 실행된 모든 자식 코루틴이 완료될 때까지 자동으로 기다려주므로, 더 이상 `Thread.sleep`이나 `Job.join()`과 같은 수동적인 동기화가 필요 없다.

### **실행의 지휘자: TestDispatcher**

어떻게 `runTest`가 시간을 마음대로 조종할 수 있을까? 그 비밀은 \*\*`TestDispatcher`\*\*에 있다. 코루틴의 `Dispatcher`는 어떤 스레드에서 코드를 실행할지 결정하는 역할을 한다. `runTest`는 내부적으로 이 `TestDispatcher`를 사용하여 모든 코루틴을 스케줄링하고, 가상 시간을 제어한다.

`runTest`는 기본적으로 `StandardTestDispatcher`를 사용한다. 이 디스패처는 실행해야 할 코루틴들을 내부 큐에 넣고, 테스트 스케줄러가 가상 시간을 진행시킬 때마다 큐에 있는 코루틴들을 실행하는 방식으로 동작한다.

```kotlin
@Test
fun dispatcher_test() = runTest { // 내부적으로 StandardTestDispatcher와 TestCoroutineScheduler를 생성
    val dispatcher = this.coroutineContext[ContinuationInterceptor] as TestDispatcher

    println("Before launch: ${currentTime}ms") // virtual time: 0ms

    launch {
        println("Inside launch, before delay: ${currentTime}ms") // 0ms
        delay(1000)
        println("Inside launch, after delay: ${currentTime}ms") // 1000ms
    }

    println("After launch: ${currentTime}ms") // 0ms
    
    // runTest가 자동으로 시간을 진행시켜주기 때문에, launch 블록이 모두 실행됨
    // 최종 virtual time은 1000ms가 된다.
}
```

개발자가 직접 `TestDispatcher`나 스케줄러를 제어할 수도 있지만(다음 절에서 다룸), 대부분의 경우 `runTest`가 제공하는 자동 시간 진행 기능만으로도 충분하다.

결론적으로, `runTest`와 그 내부의 `TestDispatcher`는 코루틴 테스트의 두 가지 핵심 문제를 해결한다. **가상 시간을 통해 '속도'를 확보하고, 자동 동기화를 통해 '신뢰성'을 보장한다.** 이 견고한 기반 위에서, 우리는 이제 Kotlin Flow와 같은 더 복잡한 비동기 패턴을 테스트할 준비를 마쳤다.