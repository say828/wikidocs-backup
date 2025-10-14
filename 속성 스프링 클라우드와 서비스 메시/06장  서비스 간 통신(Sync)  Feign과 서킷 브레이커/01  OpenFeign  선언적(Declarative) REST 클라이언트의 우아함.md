## OpenFeign: 선언적(Declarative) REST 클라이언트의 우아함

`order-service`가 `product-service`의 API를 호출해야 할 때, 어떤 코드를 작성해야 할까요?

전통적인 Spring `RestTemplate`을 사용한다면 다음과 같은 코드가 될 것입니다.

```kotlin
// RestTemplate을 사용하는 (나쁜) 예시 👎
@Service
class OrderService(
    private val restTemplate: RestTemplate,
    // (DiscoveryClient를 주입받아 직접 주소를 찾는 로직도 필요)
) {
    fun getProductInfo(productId: Long): ProductResponseDto? {
        // 1. 서비스 디스커버리에서 'product-service'의 주소를 직접 조회
        val instances = discoveryClient.getInstances("product-service")
        val serviceUri = instances.first().uri
        val url = "$serviceUri/api/v1/products/$productId"
        
        // 2. URL을 직접 만들고, HTTP GET 요청을 보내고, 응답을 DTO로 변환...
        // 3. 에러 처리, 헤더 설정 등 모든 것을 수동으로 처리해야 함
        return restTemplate.getForObject(url, ProductResponseDto::class.java)
    }
}
```

이 코드는 장황하고, 오류가 발생하기 쉬우며, 비즈니스 로직과 인프라 로직(HTTP 통신)이 뒤섞여 있습니다.

-----

### OpenFeign: 인터페이스가 곧 API 클라이언트가 되다

Spring Cloud는 이 문제를 해결하기 위해 **OpenFeign**이라는 환상적인 솔루션을 제공합니다. Feign은 **'선언적(Declarative)' REST 클라이언트**입니다.

'선언적'이라는 말은, 개발자가 "어떻게(How)" 통신할지를 코드로 작성하는 대신, **"무엇을(What)" 원하는지만 인터페이스(Interface)로 선언**하면, Feign이 런타임에 실제 HTTP 통신 코드를 **자동으로 생성**해준다는 의미입니다.

이는 마치 `Spring Data JPA`에서 `JpaRepository` 인터페이스만 선언하면 실제 DB 쿼리 구현체를 자동으로 만들어주는 것과 동일한 원리입니다.

-----

### `order-service`에 Feign 클라이언트 구현하기

#### 1\. 의존성 추가

먼저, \*\*호출하는 쪽(Client)\*\*인 `order-service`의 `build.gradle.kts`에 OpenFeign 의존성을 추가합니다.

```kotlin
// order-service/build.gradle.kts
dependencies {
    // 1. OpenFeign 스타터
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    // ...
}
```

#### 2\. Feign 클라이언트 활성화

`order-service`의 메인 애플리케이션 클래스에 `@EnableFeignClients` 어노테이션을 추가하여 Feign 기능이 활성화됨을 알립니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/OrderApplication.kt
package com.ecommerce.order

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.cloud.openfeign.EnableFeignClients

@SpringBootApplication
@EnableFeignClients // Feign 클라이언트 기능을 활성화
class OrderApplication

fun main(args: Array<String>) {
    runApplication<OrderApplication>(*args)
}
```

#### 3\. Feign 클라이언트 인터페이스 작성

이제 `order-service` 내부에 **`product-service`를 호출하기 위한 인터페이스**를 정의합니다. 이 인터페이스가 Feign의 마법이 일어나는 곳입니다.

먼저, `product-service`가 반환할 응답 DTO를 `order-service`에도 동일하게 정의합니다. (이 DTO는 서비스 간의 'API 계약'입니다.)

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/client/ProductDto.kt
package com.ecommerce.order.client

// product-service의 응답 DTO와 동일한 구조
data class ProductResponse(
    val id: Long,
    val name: String,
    val price: Long,
    val stockQuantity: Int,
)
```

이제 실제 Feign 클라이언트 인터페이스를 작성합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/client/ProductServiceClient.kt
package com.ecommerce.order.client

import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody

// 1. @FeignClient 어노테이션
// name: 호출할 서비스의 Eureka/Consul 등록 이름 (서비스 ID)
@FeignClient(name = "product-service")
interface ProductServiceClient {

    // 2. 호출할 'product-service'의 API 시그니처와 동일하게 메서드를 선언
    // Spring MVC 어노테이션을 그대로 사용
    @GetMapping("/api/v1/products/{productId}")
    fun getProduct(@PathVariable productId: Long): ProductResponse
    
    // (예시) 상품 재고 차감을 위한 API 호출
    // @PostMapping("/api/v1/products/decrease-stock")
    // fun decreaseStock(@RequestBody request: DecreaseStockRequest)
}
```

  * **`@FeignClient(name = "product-service")`**: 이 한 줄이 마법의 핵심입니다. Feign은 이 이름을 보고 **서비스 디스커버리(Eureka/Consul)에 문의**하여 `product-service`의 실제 IP 주소를 찾고, 로드 밸런싱까지 자동으로 처리합니다.

#### 4\. `OrderService`에서 Feign 클라이언트 사용

이제 `OrderService`에서 `RestTemplate` 대신 방금 만든 `ProductServiceClient` 인터페이스를 주입받아 사용합니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/service/OrderService.kt
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val productServiceClient: ProductServiceClient // 1. Feign Client 주입
) {
    @Transactional
    fun createOrder(memberId: Long, request: CreateOrderRequest) {
        
        // 2. 마치 로컬 메서드를 호출하듯 간결하게 API 호출
        val product = productServiceClient.getProduct(request.productId)
        
        // 3. 재고 확인 로직
        if (product.stockQuantity < request.quantity) {
            throw IllegalArgumentException("재고가 부족합니다.")
        }

        // ... (주문 생성 로직) ...
    }
}
```

`RestTemplate`을 사용했을 때의 복잡했던 코드가, 마치 같은 프로젝트에 있는 다른 서비스를 주입받아 **평범한 메서드를 호출**하는 것처럼 간결하고 우아하게 바뀌었습니다. 코드에서 HTTP, URL, 서비스 디스커버리 같은 인프라 로직이 완전히 사라지고 오직 '비즈니스 로직'만 남았습니다.

이것이 바로 OpenFeign이 제공하는 '선언적 REST 클라이언트'의 우아함입니다. 하지만 기억해야 할 것은, Feign은 통신을 '편하게' 만들어 줄 뿐, 06장 00절에서 논의했던 동기 통신의 '위험성' 자체를 해결해주지는 않는다는 점입니다. 다음 절에서는 이 위험을 관리하는 방법을 배웁니다.