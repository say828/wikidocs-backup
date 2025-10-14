## 소비자 주도 계약 테스트(Consumer-Driven Contract Testing): Spring Cloud Contract

단위 테스트와 통합 테스트는 '내 서비스'가 올바르게 동작하는지를 검증하는 훌륭한 방법입니다. 하지만 다음과 같은 질문에는 답하지 못합니다.

> **"내가 의존하는 `product-service` 팀이 API 응답 명세(Spec)를 나에게 말도 없이 바꾼다면 어떻게 될까?"**

`order-service`의 단위 테스트는 가짜(Mock) `ProductServiceClient`를 사용하므로, 실제 `product-service`의 API가 변경되어도 항상 통과할 것입니다. 결국 운영 환경에 배포되고 나서야 두 서비스 간의 연동이 깨졌음을 알게 되고, 이는 심각한 장애로 이어집니다.

이 문제를 해결하기 위해, 우리는 \*\*소비자 주도 계약 테스트(Consumer-Driven Contract Testing, CDC 테스트)\*\*라는 강력한 패턴을 도입합니다.

-----

### 계약 테스트란 무엇인가?

CDC 테스트의 핵심 아이디어는, 서비스 간의 직접적인 통합 테스트 대신 \*\*'계약(Contract)'\*\*이라는 가벼운 문서를 중심으로 협업하는 것입니다.

  * **소비자 (Consumer):** API를 호출하는 서비스 (예: `order-service`)
  * **제공자 (Provider):** API를 제공하는 서비스 (예: `product-service`)

**흐름:**

1.  **[소비자]** `order-service` 팀은 `product-service`에게 \*\*"나는 당신의 API로부터 이런 요청을 보냈을 때, 이런 응답이 올 것이라고 기대합니다"\*\*라는 내용의 '계약서'를 작성합니다.
2.  **[소비자]** `order-service`는 테스트 시, 실제 `product-service`를 띄우는 대신, 이 '계약서'를 기반으로 생성된 \*\*가짜 서버(Stub)\*\*를 띄워 자신의 로직을 검증합니다.
3.  **[제공자]** `product-service` 팀은 테스트 시, 소비자가 작성한 이 '계약서'를 가져와 **자신의 실제 API가 계약서의 내용과 일치하는지를 자동으로 검증**합니다.

만약 `product-service`가 API를 변경하여 '계약'을 위반하게 되면, `product-service`의 빌드 파이프라인에서 테스트가 실패하고 배포가 차단됩니다. 이를 통해 **API 호환성이 깨지는 변경이 운영 환경에 배포되는 것을 원천적으로 방지**할 수 있습니다.

-----

### Spring Cloud Contract를 이용한 구현

Spring Cloud는 이 CDC 테스트를 위한 **Spring Cloud Contract** 프레임워크를 제공합니다.

#### 1\. 소비자 측 (`order-service`): 계약서 작성 및 Stub 기반 테스트

**1-1. 의존성 추가:**
`order-service`의 `build.gradle.kts`에 `spring-cloud-starter-contract-stub-runner`를 추가합니다.

**1-2. 계약서 작성 (`Groovy DSL`):**
`test/resources/contracts` 디렉토리 아래에 계약 파일을 작성합니다.

```groovy
// order-service/src/test/resources/contracts/product/getProductById.groovy
package product

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    // 이 계약의 설명
    description "ID로 단일 상품 정보를 조회했을 때, 성공적으로 응답해야 한다"

    request { // 1. 소비자가 보낼 요청 정의
        method 'GET'
        url '/api/v1/products/1'
    }
    response { // 2. 제공자가 반환해야 할 응답 정의
        status 200
        headers {
            contentType('application/json')
        }
        body( // 3. 응답 본문 구조와 값 정의
            id: 1L,
            name: "Test Product",
            price: 10000L,
            stockQuantity: 10
        )
    }
}
```

**1-3. Stub Runner를 이용한 통합 테스트:**
`@AutoConfigureStubRunner`를 사용하여, `product-service`의 계약을 기반으로 한 가짜 서버(Stub)를 띄우고 테스트합니다.

```kotlin
// order-service/src/test/kotlin/com/ecommerce/order/client/ProductServiceClientTest.kt

@SpringBootTest
@AutoConfigureStubRunner(
    // 1. product-service의 stub을 찾아서 실행
    // stubsMode = StubRunnerProperties.StubsMode.LOCAL : 로컬 .m2 저장소에서 찾음
    ids = ["com.ecommerce:product-service:+:stubs:8082"] 
)
class ProductServiceClientTest(
    private val productServiceClient: ProductServiceClient // 실제 Feign 클라이언트
) : DescribeSpec({
    describe("getProduct") {
        it("계약에 따라 상품 정보를 성공적으로 조회한다") {
            // When: Feign 클라이언트 호출
            // @AutoConfigureStubRunner가 Feign의 URL을 localhost:8082(Stub 서버)로 자동 변경
            val product = productServiceClient.getProduct(1L)
            
            // Then: 계약서에 명시된 값과 일치하는지 검증
            product.name shouldBe "Test Product"
            product.price shouldBe 10000L
        }
    }
})
```

-----

#### 2\. 제공자 측 (`product-service`): 계약 검증

**2-1. 의존성 및 플러그인 추가:**
`product-service`의 `build.gradle.kts`에 `spring-cloud-starter-contract-verifier`와 `spring-cloud-contract-spec` 의존성, 그리고 `spring-cloud-contract` gradle 플러그인을 추가합니다. 이 플러그인은 계약서로부터 테스트 코드를 **자동 생성**하고, 테스트가 통과하면 다른 프로젝트에서 사용할 수 있는 **stub-jar를 생성**하는 역할을 합니다.

**2-2. 계약서 기반 테스트 자동 실행:**
`product-service`를 빌드(`test` 태스크 실행)하면, Spring Cloud Contract 플러그인이 다음 작업을 수행합니다.

1.  지정된 위치(예: `order-service`가 발행한 stub-jar)에서 모든 `.groovy` 계약 파일을 읽어옵니다.
2.  각 계약 파일에 대해 **JUnit 테스트 코드를 자동으로 생성**합니다.
3.  자동 생성된 테스트는 계약서의 `request`를 실제 `ProductController`로 보내고, 실제 `response`가 계약서의 `response`와 일치하는지 검증합니다.

만약 `ProductController`가 반환하는 `ProductResponse` DTO에서 `price` 필드를 `salePrice`로 변경한다면, 이 자동 생성된 테스트는 실패하고 `product-service`의 빌드가 중단됩니다. **계약을 위반하는 변경이 원천 차단된 것입니다\!**

-----

이 계약 테스트를 CI/CD 파이프라인에 통합하면, 서비스 간의 API 호환성을 항상 보장하면서도 각 팀은 서로의 서비스 실행에 의존하지 않고 독립적으로, 그리고 자신감 있게 개발 및 배포를 진행할 수 있습니다. 이는 MSA 환경에서 개발 속도와 안정성을 동시에 잡는 핵심 전략입니다.