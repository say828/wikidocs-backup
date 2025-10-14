## 비동기 MSA 구현: Feign 대신 WebClient와 코루틴으로 서비스 호출하기

12장 04절까지 우리는 컨트롤러, 서비스, 데이터베이스 접근에 이르는 수직적인 스택을 모두 논블로킹으로 전환했습니다. 하지만 MSA는 수평적인 서비스 간의 통신이 빈번하게 일어납니다. 이 마지막 퍼즐 조각이 아직 블로킹으로 남아있습니다.

우리가 06장에서 사용했던 **OpenFeign**은, 기본적으로 내부에서 **블로킹(Blocking) HTTP 클라이언트**를 사용합니다. 즉, 코루틴으로 무장한 `order-service`가 Feign을 통해 `product-service`를 호출하는 순간, **스레드는 블로킹**되어 우리가 지금까지 쌓아 올린 논블로킹의 이점이 모두 사라집니다.

-----

### WebClient: 스프링의 논블로킹 HTTP 클라이언트

이 문제를 해결하기 위한 스프링의 공식적인 해답은 \*\*`WebClient`\*\*입니다. `WebClient`는 `Spring WebFlux`에 포함된 최신 논블로킹, 리액티브 HTTP 클라이언트입니다. `RestTemplate`의 리액티브 버전이라고 생각하면 쉽습니다.

`WebClient` 자체는 `Mono`와 `Flux`를 반환하는 리액티브 API를 가지고 있어, 그대로 사용하면 12장 01절에서 겪었던 `flatMap`의 복잡성을 다시 마주하게 됩니다.

하지만 코틀린 코루틴과 만나면 마법이 일어납니다. 스프링은 `WebClient`를 `suspend` 함수 내에서 마치 블로킹 코드처럼 사용할 수 있게 해주는 강력한 \*\*코루틴 확장 함수(Coroutine Extension Function)\*\*를 제공합니다.

-----

### `order-service`의 Feign 클라이언트를 `WebClient`로 전환하기

#### 1\. 의존성 및 설정 변경

`order-service`의 `build.gradle.kts`에서 OpenFeign 의존성을 제거하고, 메인 애플리케이션 클래스의 `@EnableFeignClients` 어노테이션도 삭제합니다.

이제 `WebClient`가 05장에서 구축한 서비스 디스커버리(Eureka/Consul)와 연동되도록 `@LoadBalanced` 설정을 추가합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/config/WebClientConfig.kt
package com.ecommerce.order.config

import org.springframework.cloud.client.loadbalancer.LoadBalanced
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.reactive.function.client.WebClient

@Configuration
class WebClientConfig {

    /**
     * @LoadBalanced 어노테이션을 붙여주면,
     * 이 WebClient.Builder가 서비스 디스커버리를 통해
     * 서비스 이름을 실제 IP 주소로 변환해주는 기능이 활성화됩니다.
     */
    @Bean
    @LoadBalanced
    fun webClientBuilder(): WebClient.Builder {
        return WebClient.builder()
    }
}
```

#### 2\. `WebClient` 기반의 새로운 `ProductServiceClient` 구현

기존의 `ProductServiceClient` 인터페이스를, `WebClient`를 사용하는 \*\*구체 클래스(Concrete Class)\*\*로 변경합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/client/ProductServiceClient.kt
package com.ecommerce.order.client

import kotlinx.coroutines.reactor.awaitSingleOrNull
import org.springframework.stereotype.Component
import org.springframework.web.reactive.function.client.WebClient
import org.springframework.web.reactive.function.client.awaitBody

@Component
class ProductServiceClient(
    webClientBuilder: WebClient.Builder // 1. @LoadBalanced 설정된 Builder 주입
) {
    // 2. 서비스 이름을 기반으로 baseUrl을 설정한 WebClient 인스턴스 생성
    private val webClient = webClientBuilder.baseUrl("http://product-service").build()

    /**
     * 코루틴 확장 함수인 .awaitBody()를 사용하여 논블로킹으로 호출
     */
    suspend fun getProduct(productId: Long): ProductResponse? {
        return try {
            webClient.get()
                .uri("/api/v1/products/{productId}", productId) // 3. 호출할 URI
                .retrieve() // 4. 요청 실행 및 응답 수신
                .awaitBody<ProductResponse>() // 5. 응답 body를 DTO로 변환. 여기서 코루틴이 일시 중단!
        } catch (e: Exception) {
            // (에러 처리: 404 Not Found 등... )
            null
        }
    }
}
```

  * **`.awaitBody<T>()`**: 이 코루틴 확장 함수가 마법의 핵심입니다. HTTP 요청을 보낸 뒤, 응답이 올 때까지 스레드를 블로킹하는 대신 \*\*코루틴을 일시 중단(suspend)\*\*시킵니다. 응답이 도착하면 중단되었던 지점부터 코루틴을 재개합니다.

#### 3\. 서비스 계층 (변경 없음\!)

가장 아름다운 부분입니다. 우리의 `OrderService`는 이미 `suspend` 함수로 설계되어 있고, `ProductServiceClient`의 `getProduct` 메서드를 `suspend` 함수로서 호출하고 있었습니다.

따라서 Feign 기반 클라이언트가 `WebClient` 기반 클라이언트로 내부 구현이 완전히 바뀌었음에도 불구하고, `OrderService`의 코드는 **단 한 줄도 수정할 필요가 없습니다.**

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/service/OrderService.kt
@Service
class OrderService(
    private val orderRepository: CoroutineOrderRepository, // R2DBC
    private val productServiceClient: ProductServiceClient // WebClient 기반 구현체
) {

    suspend fun createOrder(command: CreateOrderCommand): Long {
        // 이 코드는 Feign을 쓰든 WebClient를 쓰든 동일합니다.
        // productServiceClient.getProduct()이 suspend 함수라는 계약은 변하지 않았기 때문입니다.
        val product = productServiceClient.getProduct(command.productId)
            ?: throw IllegalArgumentException("상품 정보를 찾을 수 없습니다.")

        // ...
    }
}
```

-----

이것으로 12장을 마칩니다. 우리는 마이크로서비스 아키텍처의 **모든 I/O 경로**를 완벽한 논블로킹으로 전환했습니다.

  * **외부 HTTP 요청/응답:** `Spring WebFlux` + `suspend` 컨트롤러
  * **내부 DB 통신:** `Spring Data R2DBC` + `CoroutineCrudRepository`
  * **서비스 간 통신:** `WebClient` + 코루틴 확장 함수

`Spring WebFlux`와 코틀린 `코루틴`의 완벽한 조합을 통해, 우리는 리액티브 시스템의 **성능과 확장성**을 전통적인 블로킹 코드의 **단순함과 가독성**으로 누릴 수 있게 되었습니다. 이는 대규모 트래픽을 감당해야 하는 현대 MSA를 구축하는 가장 강력한 기술적 기반이 될 것입니다.