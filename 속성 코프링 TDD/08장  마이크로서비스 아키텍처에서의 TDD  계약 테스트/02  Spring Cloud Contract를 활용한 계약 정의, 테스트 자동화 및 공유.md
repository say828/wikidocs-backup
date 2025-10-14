## 02\. Spring Cloud Contract를 활용한 계약 정의, 테스트 자동화 및 공유

소비자 주도 계약 테스트의 원리를 실제 코드로 구현하는 가장 강력한 도구는 단연 \*\*Spring Cloud Contract(SCC)\*\*다. SCC는 스프링 생태계에 완벽하게 통합되어, 계약을 정의하고, 공유하며, 검증하는 전체 워크플로우를 놀라울 정도로 자동화해준다.

이 책의 모든 코드가 코틀린으로 작성된 만큼, 계약 역시 **Kotlin DSL**을 사용하여 프로젝트 전체의 언어 일관성을 유지하는 방식으로 진행한다.

### **1단계 (소비자 측): Kotlin DSL로 계약서 작성 및 테스트**

소비자인 '주문 서비스'는 자신이 필요한 API의 모습을 코틀린 코드로 직접 작성한다.

#### **1-1. 의존성 및 플러그인 설정 (`order-service/build.gradle.kts`)**

```kotlin
plugins {
    // ...
    id("org.springframework.cloud.contract") version "4.1.1"
}

dependencies {
    // ...
    // SCC Verifier는 계약을 정의하고, 이를 기반으로 WireMock Stub을 생성하는 기능을 제공
    testImplementation("org.springframework.cloud:spring-cloud-starter-contract-verifier")
}

// SCC 플러그인 설정
contracts {
    testFramework.set(org.springframework.cloud.contract.verifier.config.TestFramework.JUNIT5)
    // 계약이 Kotlin DSL로 작성되었음을 명시
    contractDslLanguage.set(org.springframework.cloud.contract.verifier.config.DslLanguage.KOTLIN)
    // 생성될 Provider 측 테스트의 Base Class를 지정
    baseClassForTests.set("com.payment.ContractBaseTest")
}
```

#### **1-2. 계약 정의 (`src/test/resources/contracts/payment/ShouldCreatePayment.kt`)**

계약은 `contracts` 디렉토리 아래에 `org.springframework.cloud.contract.spec.Contract`를 상속받는 코틀린 클래스로 정의한다.

```kotlin
package contracts.payment

import org.springframework.cloud.contract.spec.Contract
import org.springframework.cloud.contract.spec.dsl.Http

class ShouldCreatePayment : Contract() {
    init {
        contract {
            description = "결제 서비스가 성공적으로 결제를 생성해야 한다"

            request {
                method = Http.POST
                url = url("/payments")
                headers {
                    contentType(applicationJson())
                }
                body(
                    "orderId" to 12345,
                    "amount" to 50000
                )
            }

            response {
                status = CREATED // 201
                headers {
                    contentType(applicationJson())
                }
                body(
                    // provider가 어떤 값을 반환해도 되는 필드는 정규식 등으로 유연하게 표현 가능
                    "paymentId" to regex("PAY-[a-zA-Z0-9]+"),
                    "status" to "COMPLETED"
                )
            }
        }
    }
}
```

#### **1-3. 소비자의 API 클라이언트 테스트**

소비자는 `@AutoConfigureStubRunner`를 사용하여 자신의 API 클라이언트를 테스트한다. 이 어노테이션은 우리가 Kotlin DSL로 정의한 계약으로부터 **WireMock Stub**을 자동으로 생성하고 실행시킨다.

```kotlin
// PaymentApiClientTest.kt (in Order Service)
@SpringBootTest
@AutoConfigureStubRunner(
    // Nexus/Artifactory에 배포된 Provider의 Stub Jar 좌표
    ids = ["com.payment:payment-service:+:stubs:8080"], 
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class PaymentApiClientTest {
    @Autowired lateinit var paymentApiClient: PaymentApiClient

    @Test
    fun `결제 생성을 요청하면, 성공 응답을 받아야 한다`() {
        // given
        val request = PaymentRequest(orderId = 12345, amount = 50000)

        // when
        // 이 호출은 실제 '결제 서비스'가 아닌, Kotlin DSL 계약 기반으로 생성된 WireMock Stub 서버로 향한다.
        val response = paymentApiClient.createPayment(request)

        // then
        response.status shouldBe "COMPLETED"
        response.paymentId shouldStartWith "PAY-"
    }
}
```

소비자는 이 테스트를 통해 자신의 클라이언트 코드가 계약에 맞는 응답을 올바르게 처리하는지 검증할 수 있다.

### **2단계 (제공자 측): 계약 이행 증명**

제공자 측에서는 소비자가 발행한 계약을 이행하는 것을 증명해야 한다. SCC 플러그인이 이 검증 과정을 대부분 자동화해준다.

#### **2-1. 의존성 및 플러그인 설정 (`payment-service/build.gradle.kts`)**

```kotlin
// ...
contracts {
    // 소비자 측에서 발행한 Stub Jar를 다운로드하도록 설정
    contractDependency {
        stringNotation.set("com.order:order-service:+:stubs")
    }
    baseClassForTests.set("com.payment.ContractBaseTest")
}
```

#### **2-2. 자동 생성 테스트를 위한 Base Class 작성**

SCC가 계약 파일을 기반으로 **`MockMvc` 테스트 코드를 자동으로 생성**할 수 있도록, 컨트롤러를 설정하는 Base Class를 작성해야 한다.

```kotlin
// ContractBaseTest.kt (in Payment Service)
@SpringBootTest
class ContractBaseTest {
    @Autowired lateinit var paymentController: PaymentController
    @MockBean lateinit var paymentService: PaymentService

    @BeforeEach
    fun setup() {
        RestAssuredMockMvc.standaloneSetup(paymentController)
        
        val response = PaymentResponse(paymentId = "PAY-random123", status = "COMPLETED")
        every { paymentService.createPayment(any()) } returns response
    }
}
```

#### **2-3. 계약 검증 실행**

제공자 측에서 `./gradlew build`를 실행하면, SCC 플러그인은 다운로드한 Stub Jar에서 계약을 읽어 `ContractBaseTest`를 상속받는 `MockMvc` 테스트를 **자동으로 생성**하고 실행한다.

만약 '결제 서비스' 개발자가 실수로 응답 DTO의 `status` 필드를 `paymentStatus`로 변경했다면, 이 자동 생성된 테스트가 실패하면서 빌드가 중단된다.

Kotlin DSL을 활용한 계약 테스트는 프로젝트 전체의 언어 일관성을 유지하며, Spring Cloud Contract의 강력한 자동화 워크플로우를 통해 서비스 간의 신뢰를 보장한다. 이는 코틀린을 주력으로 사용하는 현대적인 마이크로서비스 팀에게 가장 이상적인 접근법이다.