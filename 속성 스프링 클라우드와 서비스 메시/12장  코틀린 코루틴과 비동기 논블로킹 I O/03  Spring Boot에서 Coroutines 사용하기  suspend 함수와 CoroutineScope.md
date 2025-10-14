## Spring Boot에서 Coroutines 사용하기: suspend 함수와 CoroutineScope

이론은 충분합니다. 이제 실제 Spring Boot 애플리케이션에서 코루틴을 사용하여 어떻게 논블로킹 API를 만드는지 실전 코드로 알아보겠습니다.

Spring의 코루틴 지원은 **Spring WebFlux** 스택을 기반으로 합니다. 따라서, 우리의 서비스는 `spring-boot-starter-web` (Tomcat 기반) 대신 `spring-boot-starter-webflux` (Netty 기반) 의존성을 사용해야 합니다.

-----

### 1\. 프로젝트 설정 (build.gradle.kts)

기존 `member-service`를 논블로킹으로 전환한다고 가정하고, `build.gradle.kts` 파일을 수정합니다.

```kotlin
// member-service/build.gradle.kts
dependencies {
    // 1. 기존 MVC 웹 스타터 제거
    // implementation("org.springframework.boot:spring-boot-starter-web")

    // 2. WebFlux 리액티브 웹 스타터 추가
    implementation("org.springframework.boot:spring-boot-starter-webflux")

    // 3. 코루틴과 리액터(WebFlux)를 연결해주는 핵심 라이브러리
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")

    // ... (기존 jpa, kafka 등) ...
    // (JPA는 블로킹이므로, 다음 절에서 R2DBC로 교체할 예정)
}
```

-----

### 2\. `suspend` 키워드: 컨트롤러와 서비스의 마법

Spring WebFlux는 코틀린 코루틴을 1급 시민(First-class citizen)으로 지원합니다. 이는 개발자가 `Mono`나 `Flux`를 직접 다루지 않고도, **메서드에 `suspend` 키워드를 붙이는 것만으로** 논블로킹 컨트롤러를 만들 수 있다는 의미입니다.

03장에서 만들었던 `MemberController`와 `MemberService`를 코루틴 기반으로 바꿔봅시다.

#### `suspend` 컨트롤러

```kotlin
// member-service/src/main/kotlin/com/ecommerce/member/api/MemberController.kt

@RestController
@RequestMapping("/api/v1/members")
class MemberController(
    private val memberService: MemberService
) {

    @PostMapping("/join")
    // 1. suspend 키워드 추가. 반환 타입은 Mono/Flux가 아닌 일반 DTO.
    suspend fun join(@RequestBody request: JoinMemberRequest): MemberResponse {
        return memberService.join(request)
    }

    @GetMapping("/{memberId}")
    // 2. Spring이 알아서 코루틴 컨텍스트에서 이 함수를 실행하고,
    //    결과를 논블로킹 HTTP 응답으로 변환해 줍니다.
    suspend fun getMember(@PathVariable memberId: Long): MemberResponse {
        return memberService.getMember(memberId)
    }
}
```

#### `suspend` 서비스

컨트롤러가 `suspend` 함수이므로, 컨트롤러가 호출하는 서비스 메서드 또한 `suspend` 함수여야 합니다.

```kotlin
// member-service/src/main/kotlin/com/ecommerce/member/service/MemberService.kt

@Service
class MemberService(
    private val memberRepository: MemberRepository // (다음 절에서 R2DBC 리포지토리로 변경 예정)
) {

    suspend fun join(request: JoinMemberRequest): MemberResponse {
        // ... (이메일 중복 검사 등) ...
        
        // 3. 이 메서드가 호출하는 DB 접근 코드 또한 suspend 함수여야 함
        val newMember = memberRepository.save(request.toEntity()) // (가정) save가 suspend 함수
        return newMember.toResponse()
    }

    suspend fun getMember(memberId: Long): MemberResponse {
        val member = memberRepository.findById(memberId) // (가정) findById가 suspend 함수
            ?: throw EntityNotFoundException("...")
        return member.toResponse()
    }
}
```

놀랍게도, **코드는 `@Transactional`이 빠지고 `suspend` 키워드가 추가된 것을 제외하면 블로킹 버전과 거의 동일**합니다. Spring WebFlux가 내부적으로 `suspend` 함수를 `Mono`로 변환하고 리액티브 파이프라인에 연결하는 모든 복잡한 작업을 대신 처리해주기 때문입니다.

-----

### 3\. `CoroutineScope`: 백그라운드 작업 실행하기

`suspend` 함수는 다른 `suspend` 함수 안에서만 호출될 수 있습니다. 만약 일반적인 함수(non-suspend)에서 비동기 작업을 시작하고 싶거나, 현재 요청-응답 흐름과 관계없는 **별개의 백그라운드 작업**을 실행하고 싶다면 어떻게 해야 할까요?

이때 \*\*`CoroutineScope`\*\*와 \*\*코루틴 빌더(Coroutine Builder)\*\*를 사용합니다.

  * **`CoroutineScope`**: 코루틴이 실행되는 '범위' 또는 '컨텍스트'를 정의합니다.
  * **`launch`**: 결과를 반환하지 않는 "실행 후 잊어버리는(Fire-and-forget)" 방식의 코루틴을 시작합니다.
  * **`async`**: 결과를 반환하는(`Deferred<T>`) 코루틴을 시작하고, 나중에 `await()`를 통해 결과를 얻을 수 있습니다.

**시나리오:** `order-service`에서 주문이 생성된 후, 현재 API 응답을 블로킹하지 않고 별도의 코루틴으로 '감사 로그(Audit Log)'를 비동기적으로 남기기

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/service/OrderService.kt
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch

@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val auditLogService: AuditLogService
) {
    // 1. 서비스에 적합한 CoroutineScope를 정의.
    //    Dispatchers.IO는 I/O 작업에 최적화된 스레드 풀을 사용.
    private val scope = CoroutineScope(Dispatchers.IO)

    suspend fun createOrder(command: CreateOrderCommand): Long {
        val savedOrder = orderRepository.save(...) // 1. 주문 생성 (suspend)

        // 2. 현재 요청-응답 흐름을 막지 않고, 별도의 코루틴으로 백그라운드 작업 실행
        scope.launch {
            auditLogService.record(
                action = "ORDER_CREATED",
                details = "Order ${savedOrder.id} created by member ${command.memberId}"
            ) // auditLogService.record는 suspend 함수
        }
        
        // 3. 감사 로그 작업이 끝나기를 기다리지 않고 즉시 응답 반환
        return savedOrder.id!!
    }
}

// audit-log-service/src/main/kotlin/com/ecommerce/audit/AuditLogService.kt
@Service
class AuditLogService {
    // 감사 로그를 DB나 로그 파일에 비동기적으로 기록하는 suspend 함수
    suspend fun record(action: String, details: String) {
        // ... (논블로킹 DB 쓰기 또는 파일 쓰기 로직) ...
        delay(100) // (예시) I/O 작업을 흉내 내기 위한 지연
        log.info("[Audit] Action: {}, Details: {}", action, details)
    }
}
```

이처럼 Spring Boot와 코루틴을 함께 사용하면, `suspend` 키워드만으로 가독성 높은 논블로킹 코드를 작성할 수 있으며, `CoroutineScope`를 통해 복잡한 동시성 처리까지 깔끔하게 구현할 수 있습니다.

이제 마지막 퍼즐 조각이 남았습니다. 어떻게 우리의 **데이터베이스 접근**까지 완벽한 논블로킹으로 만들 수 있을까요? 그 해답이 바로 다음 절의 `R2DBC`입니다.