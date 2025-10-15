## 04\. Coroutines와 JPA: 블로킹(Blocking) I/O의 한계와 `Dispatchers.IO` 활용

코틀린의 세계에 깊이 들어갈수록, 우리는 동시성 프로그래밍의 표준으로 자리 잡은 \*\*코루틴(Coroutines)\*\*과 마주하게 된다. 코루틴은 적은 수의 스레드로 수많은 동시 작업을 효율적으로 처리할 수 있게 해주는 비동기(asynchronous), 논블로킹(non-blocking) 프로그래밍의 강력한 패러다임이다.

하지만 여기서 JPA의 근본적인 한계와 코루틴의 철학이 정면으로 충돌하는 지점이 발생한다.

-----

### **근본적인 충돌: 블로킹 vs. 논블로킹**

  * **JPA/JDBC의 세계 (블로킹)**: JPA가 내부적으로 사용하는 JDBC(Java Database Connectivity) API는 근본적으로 **블로킹(Blocking) I/O** 모델을 기반으로 한다. `repository.findById(1L)`를 호출하면, 해당 쿼리를 실행하는 스레드는 데이터베이스가 응답을 줄 때까지 **그 자리에서 멈춰서 기다린다(blocked).** 스레드는 그 시간 동안 다른 어떤 일도 할 수 없다.

  * **코루틴의 세계 (논블로킹)**: 코루틴은 스레드를 절대 막지 않는 것을 원칙으로 한다. I/O 작업처럼 오래 걸리는 일을 만나면, 코루틴은 스레드를 막는 대신 자신을 \*\*'일시 중단(suspend)'\*\*시키고, 해당 스레드는 다른 코루틴의 작업을 처리하도록 양보한다. I/O 작업이 끝나면, 코루틴은 다시 스레드 풀의 가용한 스레드에서 작업을 재개한다.

만약 코루틴 안에서 아무런 처리 없이 JPA 코드를 그대로 호출하면 어떻게 될까? `suspend` 함수 안에서 `memberRepository.findById()`를 호출하는 순간, 코루틴의 원칙이 깨지고 **실행 중인 스레드가 그대로 블로킹**된다. 이는 코루틴을 사용하는 의미를 퇴색시키고, 심각한 성능 저하와 스레드 고갈(Thread Starvation) 문제로 이어질 수 있다.

-----

### **해결책: 블로킹 코드를 `Dispatchers.IO`로 격리하기**

이 문제를 해결하기 위해 코틀린 코루틴은 **`Dispatchers.IO`** 라는 특별한 '실행 컨텍스트(Dispatcher)'를 제공한다. `Dispatchers.IO`는 블로킹 I/O 작업을 처리하기 위해 특별히 설계된 스레드 풀이다.

코루틴 내에서 JPA와 같은 블로킹 코드를 호출하는 올바른 방법은, 해당 호출 부분을 **`withContext(Dispatchers.IO) { ... }`** 블록으로 감싸주는 것이다.

**`suspend` 함수 내에서의 올바른 JPA 호출 패턴**

```kotlin
@Service
class MemberService(
    private val memberRepository: MemberRepository
) {
    // Coroutine Scope 내에서 실행되는 suspend 함수
    suspend fun findMemberDtoAsync(id: Long): MemberDto {
        // withContext를 사용하여 블로킹 호출을 IO 스레드 풀로 보낸다.
        return withContext(Dispatchers.IO) {
            println("Executing on thread: ${Thread.currentThread().name}") // "DefaultDispatcher-worker-..."
            
            // 이 블록 안에서 실행되는 JPA 코드는 IO Dispatcher의 스레드를 블로킹한다.
            // 하지만 이 함수를 호출한 원래 스레드(예: UI 스레드)는 막히지 않는다.
            val member = memberRepository.findById(id).orElseThrow()
            
            // DB 작업이 끝난 후 DTO로 변환하여 반환
            MemberDto(member.username, member.team.name)
        }
    }
}
```

`withContext(Dispatchers.IO)`는 현재 코루틴의 실행을 잠시 `Dispatchers.IO`의 스레드로 전환시킨다. 그 안에서 `findById()`라는 블로킹 작업이 안전하게 실행되는 동안, 원래의 스레드는 다른 일을 할 수 있다. DB 작업이 끝나면, 결과와 함께 다시 원래의 디스패처로 복귀한다.

이 패턴은 JPA 코드를 비동기적으로 만드는 것이 아니라, **블로킹 작업을 안전하게 격리**하는 '다리' 역할을 한다.

-----

> **한계와 미래: R2DBC**
> `withContext(Dispatchers.IO)`는 블로킹 라이브러리를 코루틴 세상에서 사용하기 위한 실용적인 '해결책'이지만, 근본적인 '해법'은 아니다. 데이터베이스 호출 자체가 논블로킹으로 동작하는 것은 아니기 때문이다.
>
> 진정한 논블로킹 관계형 데이터베이스 접근을 위해서는 JDBC를 대체하는 **R2DBC(Reactive Relational Database Connectivity)** 라는 새로운 표준이 필요하다. 하지만 JPA는 JDBC 위에 구축된 기술이므로 R2DBC와는 호환되지 않는다. 완전한 리액티브(Reactive) 스택을 구축하려면, JPA를 포기하고 Spring Data R2DBC나 jasync-sql, Exposed 같은 다른 라이브러리를 사용해야 하는 큰 아키텍처적 결정을 내려야 한다.
>
> -----

이것으로 코틀린의 특성을 활용하여 JPA 코드를 더 안전하고 표현력 있게 만드는 여정을 마쳤다. 우리는 `data class`를 이용한 안전한 수정 패턴부터 시작하여, `null-safety`와 `all-open` 플러그인을 통한 안정성 확보, 확장 함수를 통한 코드 개선, 그리고 코루틴과의 공존 방법까지 탐구했다.

지금까지 14장에 걸쳐 우리는 JPA라는 거대한 산의 거의 모든 봉우리를 정복했다. 이제 마지막 15장에서는 산 정상에서 우리가 걸어온 길을 돌아보고, JPA의 현재와 미래, 그리고 그 너머에 있는 새로운 기술의 지평선을 조망하며 이 긴 여정을 마무리하고자 한다.