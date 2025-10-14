## 02\. 장애 주입(Fault Injection)과 회복탄력성(Resilience) 테스트: Chaos Engineering의 원리

성능 좋은 시스템을 만드는 것만으로는 충분하지 않다. 실제 운영 환경은 예측 불가능한 장애로 가득 차 있다. 데이터베이스가 갑자기 응답하지 않고, 네트워크 지연 시간이 급증하며, 외부 API가 에러를 반환하는 상황은 '만약'의 문제가 아니라 '언제'의 문제다. \*\*회복탄력성(Resilience)\*\*이란 바로 이러한 예측 불가능한 장애 상황 속에서도 시스템이 핵심 기능을 유지하고, 우아하게 실패하며, 스스로 복구할 수 있는 능력을 의미한다.

그렇다면 이 '회복탄력성'을 어떻게 TDD로 검증할 수 있을까? "장애가 발생했을 때, 우리 시스템은 살아남아야 한다"는 요구사항을 테스트로 만드는 것, 이것이 바로 \*\*카오스 엔지니어링(Chaos Engineering)\*\*의 원리를 TDD에 접목하는 것이다. 우리는 의도적으로 시스템에 장애를 주입(Fault Injection)하고, 그 혼돈(Chaos) 속에서 시스템이 우리가 기대하는 대로 동작하는지를 검증한다.

### **회복탄력성 TDD 워크플로우**

카오스 엔지니어링의 TDD는 시스템의 '불행한 경로(unhappy path)'를 먼저 명세하는 과정이다.

1.  **RED (실패하는 카오스 테스트 작성):** 먼저, 특정 장애 시나리오와 그에 대한 기대 동작을 테스트 코드로 명세한다. "결제 API가 5초 동안 응답하지 않는 '타임아웃' 장애 상황에서, 주문 서비스는 즉시 실패하지 않고 3번의 재시도(Retry) 후 최종적으로 '결제 시스템 오류' 응답을 반환해야 한다."
2.  **INVESTIGATE (현재 동작 확인):** 이 테스트를 처음 실행하면, 아마도 시스템은 아무런 회복탄력성 패턴(재시도, 서킷 브레이커 등)이 없으므로 그대로 멈춰버리거나(무한 대기), 원치 않는 예외를 던지며 **실패**할 것이다.
3.  **FIX (회복탄력성 패턴 적용):** 실패를 확인한 후, **Resilience4j**, **Spring Retry**와 같은 라이브러리를 사용하거나 직접 구현하여 필요한 회복탄력성 패턴을 코드에 적용한다. 위 시나리오에서는 '재시도(Retry)' 패턴과 '타임아웃(Timeout)' 설정을 추가한다.
4.  **GREEN (기대 동작 검증):** 카오스 테스트를 다시 실행한다. 이제 시스템은 장애 상황을 인지하고, 재시도 로직을 수행한 뒤, 우리가 명세한 대로 우아하게 실패하며 테스트는 **성공**한다.

### **실전 예시: Testcontainers와 Toxiproxy를 이용한 네트워크 장애 주입**

어떻게 테스트 환경에서 "데이터베이스가 갑자기 느려지는" 상황을 시뮬레이션할 수 있을까? **Toxiproxy**는 바로 이러한 네트워크 수준의 혼돈을 만들기 위해 태어난 도구다. Testcontainers는 Toxiproxy 컨테이너를 지원하여, 우리 애플리케이션과 다른 컨테이너(예: PostgreSQL) 사이의 네트워크 연결을 가로채고 지연 시간(latency), 대역폭 제한(bandwidth limit), 연결 끊김(cut) 등의 장애를 동적으로 주입할 수 있게 해준다.

**TDD 예시: DB 지연에 대한 서킷 브레이커(Circuit Breaker) 테스트**

  * **요구사항:** "`ProductRepository`의 DB 커넥션 응답이 1초 이상 걸리는 상황이 5번 연속 발생하면, 서킷 브레이커가 열려(Open) 이후의 모든 요청은 DB를 호출하지 않고 즉시 실패해야 한다."

**1. 실패하는 카오스 테스트 작성 (RED)**

```kotlin
// AbstractChaosDataJpaTest.kt (Toxiproxy 설정 추가)
@Testcontainers
abstract class AbstractChaosDataJpaTest {
    companion object {
        // 실제 DB 컨테이너
        @Container @JvmStatic val postgres = PostgreSQLContainer("postgres:15-alpine")
        
        // DB로 가는 네트워크를 가로챌 Toxiproxy 컨테이너
        @Container @JvmStatic val toxiproxy = ToxiproxyContainer("ghcr.io/shopify/toxiproxy:2.5.0")
            .withNetwork(postgres.network)

        @DynamicPropertySource @JvmStatic
        fun registerDataSourceProperties(registry: DynamicPropertyRegistry) {
            // 애플리케이션은 Toxiproxy를 통해 DB에 연결하도록 설정
            val proxy = toxiproxy.getProxy(postgres, 5432)
            registry.add("spring.datasource.url") { "jdbc:postgresql://${proxy.containerIpAddress}:${proxy.proxyPort}/${postgres.databaseName}" }
            // ... username, password ...
        }
    }
}

// ProductRepositoryResilienceTest.kt
class ProductRepositoryResilienceTest : AbstractChaosDataJpaTest() {
    @Test
    fun `DB 지연이 계속되면 서킷 브레이커가 열려야 한다`() {
        // given: 2초의 지연 시간을 주입하는 'latency' 프록시(toxic) 설정
        ToxiproxyClient(toxiproxy.host, toxiproxy.controlPort)
            .getProxy("postgres")
            .toxics().latency("db_latency", ToxicDirection.DOWNSTREAM, 2000)

        // when: 서킷 브레이커의 실패 임계값(5번)을 초과하여 repository 메소드 호출
        repeat(5) {
            shouldThrow<DataAccessException> { productRepository.findAll() }
        }

        // then: 6번째 호출은 DB를 호출하지 않고 서킷 브레이커에 의해 즉시 차단되어야 한다.
        // CircuitBreakerOpenException은 Resilience4j가 던지는 예외
        shouldThrow<CircuitBreakerOpenException> {
            productRepository.findAll()
        }
    }
}
```

`Resilience4j`의 서킷 브레이커 설정이 없으므로, 이 테스트는 6번째 호출에서도 계속 DB를 호출하려다 타임아웃 예외가 발생하며 **실패**한다.

**2. 회복탄력성 패턴 적용 및 검증 (GREEN)**
`ProductService`나 관련 설정 클래스에 `@CircuitBreaker` 어노테이션을 추가하고 실패 임계값을 5로 설정한다. 이제 다시 테스트를 실행하면, 5번의 실패 후 서킷이 열려 `CircuitBreakerOpenException`이 발생하므로 테스트는 **성공**한다.

-----

TDD를 통한 회복탄력성 테스트는 더 이상 장애 대응을 운영팀의 '사후 처리' 영역으로 남겨두지 않는다. 그것은 개발자가 개발 단계에서부터 시스템의 생존 전략을 코드로 명세하고 검증하는 **사전 설계 활동**이 된다. Toxiproxy와 같은 카오스 엔지니어링 도구를 활용하여, 우리는 실제 운영 환경에서 마주할 수 있는 가장 혹독한 조건 속에서도 우리 시스템이 살아남을 것이라는 강력한 자신감을 얻게 된다.