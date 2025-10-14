## 03\. 외부 API 클라이언트 테스트: WireMock을 이용한 예측 불가능한 외부 시스템 격리

애플리케이션은 홀로 동작하지 않는다. 결제 게이트웨이, 소셜 로그인, 날씨 정보, 환율 정보 등 수많은 외부 서비스 API와 통신하며 비즈니스 가치를 만들어낸다. 하지만 이러한 외부 API는 우리 테스트의 신뢰도를 저해하는 가장 큰 요인 중 하나다.

  * **느리다:** 네트워크 지연 시간은 테스트 실행 속도를 급격히 저하시킨다.
  * **불안정하다:** 외부 서비스가 다운되거나 점검 중이면 우리 테스트는 실패한다. 우리 코드에는 아무런 문제가 없음에도 말이다.
  * **비용이 든다:** 실제 과금이 발생하는 유료 API를 테스트마다 호출할 수는 없다.
  * **결과를 예측할 수 없다:** 환율처럼 값이 계속 변하거나, 특정 상황(예: 오류 응답)을 재현하기 어렵다.

이러한 불확실성을 제거하고 **FIRST** 원칙을 지키기 위해, 우리는 실제 API 서버 대신 **Mock API 서버**를 사용해야 한다. 그리고 이 분야의 업계 표준은 단연 **WireMock**이다.

WireMock은 테스트 중에만 실행되는 경량 HTTP 서버다. 우리는 이 서버에 "만약 `/payments/TX-123` 경로로 GET 요청이 오면, 이 JSON 본문과 200 OK 상태 코드로 응답해" 와 같이 규칙(Stub)을 미리 설정해 둘 수 있다. 그러면 우리 애플리케이션의 API 클라이언트는 실제 외부 서버가 아닌, 완벽하게 통제된 WireMock 서버와 통신하게 된다.

### **전문가의 선택: Testcontainers 위에서 WireMock 실행하기**

WireMock은 라이브러리 형태로 직접 실행할 수도 있지만, 우리가 앞서 배운 Testcontainers의 철학을 일관되게 적용하는 것이 가장 좋다. WireMock을 도커 컨테이너로 실행하면, 그 버전과 설정이 모든 개발 환경과 CI에서 완벽하게 동일함을 보장하여 테스트의 재현성을 극대화할 수 있다.

#### **1. WireMock Testcontainer 설정 (`AbstractIntegrationTest`)**

데이터베이스, Kafka와 마찬가지로 모든 통합 테스트가 공유할 수 있는 추상 기본 클래스를 만든다.

```kotlin
// AbstractIntegrationTest.kt (테스트 소스셋)
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.GenericContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers

@SpringBootTest
@Testcontainers
abstract class AbstractIntegrationTest {

    companion object {
        // WireMock 공식 도커 이미지를 사용한다.
        @Container
        @JvmStatic
        val wireMockContainer = GenericContainer("wiremock/wiremock:3.3.1")
            .withExposedPorts(8080)

        @DynamicPropertySource
        @JvmStatic
        fun registerApiServerProperties(registry: DynamicPropertyRegistry) {
            // 우리 API 클라이언트가 바라볼 외부 서버의 base-url을
            // Testcontainer 위에서 실행된 WireMock의 동적 주소로 덮어쓴다.
            val baseUrl = "http://localhost:${wireMockContainer.getMappedPort(8080)}"
            registry.add("external.payment-api.base-url", { baseUrl })
        }
    }
}
```

#### **2. TDD for 외부 API 클라이언트 (`PaymentApiClient`)**

외부 결제 시스템 API를 호출하여 거래 상태를 조회하는 `PaymentApiClient`를 TDD로 개발해보자.

```kotlin
// PaymentApiClientTest.kt
import com.github.tomakehurst.wiremock.client.WireMock.*
import com.github.tomakehurst.wiremock.junit5.WireMockTest
import org.springframework.beans.factory.annotation.Autowired

// WireMock 클라이언트 설정을 위해 @WireMockTest 어노테이션을 사용할 수 있다.
@WireMockTest(httpPort = 8080) // 로컬 실행 시 포트를 고정할 수도 있다.
class PaymentApiClientTest(
    @Autowired private val paymentApiClient: PaymentApiClient
) : AbstractIntegrationTest(), BehaviorSpec({

    given("외부 결제 API가 특정 거래에 대해 '성공' 상태를 반환하도록 설정되었을 때") {
        val transactionId = "TX-SUCCESS-123"
        
        // WireMock Stub 설정: 특정 URL로 GET 요청이 오면,
        stubFor(get(urlEqualTo("/payments/$transactionId"))
            .willReturn(aResponse()
                .withStatus(200) // 200 OK 응답
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "id": "$transactionId",
                        "status": "COMPLETED",
                        "amount": 50000
                    }
                """.trimIndent())
            ))

        `when`("해당 거래 ID로 상태를 조회하면") {
            val response = paymentApiClient.getTransactionStatus(transactionId)

            then("API 응답이 정확히 DTO로 변환되어야 한다") {
                response.id shouldBe transactionId
                response.status shouldBe "COMPLETED"
                response.amount shouldBe 50000
            }
        }
    }
    
    given("외부 API가 404 Not Found를 반환할 때") {
        val transactionId = "TX-NOT-FOUND-456"

        stubFor(get(urlEqualTo("/payments/$transactionId"))
            .willReturn(aResponse().withStatus(404)))

        `when`("해당 거래 ID로 상태를 조회하면") {
            then("사용자 정의 예외(TransactionNotFoundException)가 발생해야 한다") {
                shouldThrow<TransactionNotFoundException> {
                    paymentApiClient.getTransactionStatus(transactionId)
                }
            }
        }
    }
})
```

**TDD 사이클:**

1.  **RED:** `PaymentApiClientTest`를 먼저 작성한다. WireMock으로 외부 API의 성공 응답과 실패 응답을 모두 미리 정의한다. `PaymentApiClient` 구현체가 없으므로 테스트는 실패한다.
2.  **GREEN:** `RestTemplate`이나 `WebClient`를 사용하여 `PaymentApiClient`를 구현한다. 설정 파일(`application.yml`)의 `external.payment-api.base-url` 값을 읽어와 HTTP 요청을 보내고, JSON 응답을 DTO로 역직렬화한다. 404 응답을 받았을 때 예외를 던지는 로직도 추가한다. 이제 모든 테스트가 통과한다.
3.  **REFACTOR:** API 클라이언트 코드의 가독성을 높이거나, 중복되는 설정이 있다면 개선한다.

-----

Testcontainers를 활용한 데이터베이스, 메시지 큐, 그리고 WireMock 기반의 Mock API 서버 테스트까지, 우리는 이제 인프라스트럭처 계층의 모든 외부 의존성을 완벽하게 통제할 수 있는 강력한 무기를 갖추었다. 더 이상 외부 시스템의 불확실성 때문에 테스트 작성을 망설일 필요가 없다.

이것으로 우리는 아키텍처의 가장 깊은 곳(도메인)에서 시작하여 가장 바깥쪽(인프라)까지 내려오는 'Inside-Out' 방식의 TDD 여정을 마쳤다. 다음 장에서는 관점을 180도 전환하여, 사용자의 시선이 가장 먼저 닿는 곳, 즉 API 엔드포인트에서부터 개발을 시작하는 **'Outside-In' TDD** 전략을 탐험하며 시스템 전체의 흐름을 완성해 나갈 것이다.