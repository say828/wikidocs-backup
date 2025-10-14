## '주문' 서비스와 '상품' 서비스 간의 API 계약 검증

14장 03절의 이론을 바탕으로, `order-service`(소비자)와 `product-service`(제공자) 사이의 API 계약을 `Spring Cloud Contract`로 검증하는 전체 과정을 실제 코드로 구현해 보겠습니다.

-----

### 1\. 소비자 측 (`order-service`): 계약 정의 및 Stub 테스트

`order-service`는 `product-service`로부터 상품 정보를 조회하는 API에 대한 '기대사항'을 계약으로 정의하고, 이 계약을 기반으로 생성된 가짜 서버(Stub)를 상대로 자신의 Feign 클라이언트가 올바르게 동작하는지 테스트합니다.

#### 1-1. 계약서 작성

`order-service`의 `test/resources/contracts/product` 디렉토리에 다음과 같이 계약서를 작성합니다.

```groovy
// order-service/src/test/resources/contracts/product/shouldReturnProductWhenExists.groovy
package product

import org.springframework.cloud.contract.spec.Contract

Contract.make {
    request {
        method 'GET'
        url '/api/v1/products/1' // productId가 1인 상품을 요청
    }
    response {
        status 200
        headers {
            contentType('application/json')
        }
        body(
            id: 1L,
            name: "Test Product from Contract", // Stub이 반환할 값
            price: 15000L,
            stockQuantity: 50
        )
    }
}
```

#### 1-2. 소비자 테스트 코드 작성

`@AutoConfigureStubRunner`를 사용하여 위 계약서에 기반한 Stub 서버를 실행하고, `ProductServiceClient`(Feign 클라이언트)가 이 Stub 서버와 올바르게 통신하는지 검증합니다.

```kotlin
// order-service/src/test/kotlin/com/ecommerce/order/client/ProductServiceClientTest.kt
package com.ecommerce.order.client

import io.kotest.core.spec.style.DescribeSpec
import io.kotest.matchers.shouldBe
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties

@SpringBootTest
// 1. Stub Runner 설정
@AutoConfigureStubRunner(
    // 2. product-service의 stub을 찾음. '+'는 최신 버전을 의미
    //    stubsMode=LOCAL은 로컬 Maven/Gradle 캐시(.m2)에서 stub을 찾음
    ids = ["com.ecommerce:product-service:+:stubs:8082"],
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class ProductServiceClientTest(
    private val productServiceClient: ProductServiceClient // 3. 실제 Feign 클라이언트 주입
) : DescribeSpec({
    describe("getProduct") {
        it("계약에 따라 상품 정보를 성공적으로 조회한다") {
            // When
            // 4. Feign 클라이언트 호출. 이 요청은 실제 product-service가 아닌,
            //    8082 포트에 떠 있는 가짜 Stub 서버로 전송됨
            val product = productServiceClient.getProduct(1L)
            
            // Then
            // 5. Stub 서버가 반환한 값이 계약서의 body 내용과 일치하는지 검증
            product.name shouldBe "Test Product from Contract"
            product.price shouldBe 15000L
            product.stockQuantity shouldBe 50
        }
    }
})
```

`order-service` 측의 CI 파이프라인은 이 테스트를 실행하여, `product-service`의 API 명세가 변경되더라도 자신의 Feign 클라이언트가 깨지지 않았음을 항상 보장받을 수 있습니다.

-----

### 2\. 제공자 측 (`product-service`): 계약 준수 검증

`product-service`는 `order-service`(소비자)가 정의한 계약을 자신의 API가 잘 준수하고 있는지 검증해야 합니다.

#### 2-1. 의존성 및 플러그인 설정

`product-service`의 `build.gradle.kts`에 `spring-cloud-starter-contract-verifier`와 gradle 플러그인을 설정합니다.

#### 2-2. 계약 검증을 위한 Base Test 클래스 작성

Spring Cloud Contract는 계약을 기반으로 테스트 코드를 '자동 생성'합니다. 하지만 자동 생성된 코드는 우리 애플리케이션의 `Controller`를 어떻게 실행하고, `Service`를 어떻게 Mocking해야 하는지 알지 못합니다. 이 '준비' 로직을 `Base` 클래스에 작성해두면, 자동 생성된 모든 테스트가 이 클래스를 상속받아 실행됩니다.

```kotlin
// product-service/src/test/kotlin/com/ecommerce/product/contract/BaseContractTest.kt
package com.ecommerce.product.contract

import com.ecommerce.product.api.ProductController
import com.ecommerce.product.api.ProductResponse
import com.ecommerce.product.service.ProductService
import io.mockk.every
import io.restassured.module.mockmvc.RestAssuredMockMvc
import org.junit.jupiter.api.BeforeEach
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.boot.test.mock.mockito.MockBean
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.setup.MockMvcBuilders

@WebMvcTest(ProductController::class) // 1. ProductController만 테스트
abstract class BaseContractTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @MockBean // 2. ProductService는 실제 DB를 사용하지 않도록 Mocking
    private lateinit var productService: ProductService

    @BeforeEach
    fun setup() {
        // 3. RestAssured가 MockMvc를 사용하도록 설정
        RestAssuredMockMvc.mockMvc(mockMvc)

        // 4. 계약 검증을 위한 Mocking 준비
        //    'getProductById.groovy' 계약은 productId가 1인 경우를 테스트하므로,
        //    productService.getProduct(1L)이 호출될 때 계약서의 body와 동일한
        //    객체를 반환하도록 설정합니다.
        every { productService.getProduct(1L) } returns ProductResponse(
            id = 1L,
            name = "Test Product from Contract",
            price = 15000L,
            stockQuantity = 50
        )
    }
}
```

#### 2-3. 빌드 및 자동 검증

이제 `product-service`에서 `./gradlew build`를 실행하면, Spring Cloud Contract 플러그인이 마법을 부립니다.

1.  `order-service`가 만들어 배포한 `order-service-contracts.jar` 파일(또는 Git 저장소)에서 `*.groovy` 계약서들을 가져옵니다.
2.  `shouldReturnProductWhenExists.groovy` 계약서를 기반으로, `BaseContractTest`를 상속받는 테스트 클래스를 `build/generated-test-sources`에 **자동으로 생성**합니다.
3.  자동 생성된 테스트는 `MockMvc`를 통해 `/api/v1/products/1`로 `GET` 요청을 보냅니다.
4.  요청은 `ProductController`로 전달되고, Mocking된 `productService`를 호출하여 `ProductResponse`를 반환합니다.
5.  자동 생성된 테스트는 이 실제 응답이, 계약서의 `response.body`와 **필드 하나하나까지 모두 일치하는지 검증**합니다.

만약 `product-service` 개발자가 실수로 `ProductResponse`의 `stockQuantity` 필드를 `stock`으로 변경했다면, 이 테스트는 즉시 실패하고 빌드가 중단됩니다. 이를 통해 **소비자와의 계약을 깨뜨리는 변경이 프로덕션에 배포되는 것을 사전에 완벽하게 차단**할 수 있습니다.