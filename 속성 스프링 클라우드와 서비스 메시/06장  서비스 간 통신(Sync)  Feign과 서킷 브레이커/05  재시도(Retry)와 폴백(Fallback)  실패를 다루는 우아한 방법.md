## 재시도(Retry)와 폴백(Fallback): 실패를 다루는 우아한 방법

06장 03절에서 우리는 `Resilience4j` 서킷 브레이커를 통해 장애가 발생한 서비스로의 호출을 차단하고 `CallNotPermittedException` 예외를 던져 시스템 전체를 보호했습니다. 이것은 매우 효과적인 장애 격리 방법이지만, 사용자 입장에서는 "오류가 발생했습니다"라는 차가운 메시지만 보게 될 뿐입니다.

이것에 대한 해답이 바로 \*\*재시도(Retry)\*\*와 **폴백(Fallback)** 패턴입니다.

-----

### 1\. 재시도 (Retry): "혹시 모르니 한 번 더"

재시도 패턴은 04장 05절(게이트웨이 안정성)에서 이미 다루었습니다. 서킷 브레이커가 '지속적인 장애'에 대응한다면, 재시도는 **'일시적인(Transient)' 네트워크 오류**나 간헐적인 서버 응답 실패에 대응하는 데 효과적입니다.

Resilience4j는 서킷 브레이커와 별개로 강력한 `Retry` 모듈을 제공합니다. `application.yml`에 다음과 같이 설정하면, Feign 호출이 실패했을 때 서킷 브레이커가 작동하기 전에 먼저 몇 차례 재시도를 수행하도록 구성할 수 있습니다.

```yaml
# order-service/src/main/resources/application.yml
resilience4j:
  retry:
    instances:
      product-service:
        max-attempts: 3 # 1. 최대 3번 시도 (첫 시도 + 2번 재시도)
        wait-duration: 100ms # 2. 재시도 사이 100ms 대기
        # 3. 어떤 예외 발생 시 재시도할 것인지 지정 (예: 5xx 서버 에러)
        retry-exceptions:
          - feign.FeignException$ServiceUnavailable
          - java.util.concurrent.TimeoutException
```

하지만 재시도만으로는 근본적인 장애 상황(서비스 다운, 지속적인 성능 저하)을 해결할 수 없습니다. 재시도 횟수를 모두 소진하고 결국 실패하게 되면, 서킷 브레이커가 개입할 차례입니다.

-----

### 2\. 폴백 (Fallback)

**폴백**은 주(Primary) 로직(API 호출)이 실패했을 때, 실행되는 **대체(Alternative) 로직**을 의미합니다. 서킷 브레이커가 `OPEN` 상태가 되어 `CallNotPermittedException`을 던지는 대신, 미리 정의된 '대체 응답'이나 '대체 동작'을 수행하는 것입니다.

**레스토랑 비유:**

  * **기존:** '그릴 스테이션'에 문제가 생기면, 웨이터는 손님에게 "주문 실패했습니다"라고 말합니다.
  * **폴백 적용:** '그릴 스테이션'에 문제가 생기면, 웨이터는 손님에게 \*\*"죄송합니다. 현재 그릴 요리는 불가능하지만, 대신 오늘의 특선 스프를 바로 준비해 드릴 수 있습니다"\*\*라고 제안합니다.

손님은 원했던 스테이크는 못 먹었지만, 굶지 않고 스프라도 먹을 수 있게 되었습니다. 완벽한 서비스는 아니지만, **'서비스가 중단되는(Downtime)' 최악의 상황은 피했습니다.** 이것이 바로 '우아한 실패'입니다.

-----

### OpenFeign과 Fallback 구현하기

OpenFeign은 `@FeignClient` 어노테이션의 `fallback` 속성을 통해 이 폴백 메커니즘을 매우 간단하게 구현할 수 있도록 지원합니다.

#### 1\. Fallback 구현 클래스 작성

먼저, 원래의 Feign 클라이언트 인터페이스(`ProductServiceClient`)를 `implements`하는 **Fallback 클래스**를 만듭니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/client/ProductServiceClientFallback.kt
package com.ecommerce.order.client

import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component

@Component
class ProductServiceClientFallback : ProductServiceClient {
    
    private val log = LoggerFactory.getLogger(javaClass)

    /**
     * 이 메서드는 ProductServiceClient.getProduct() 호출이 실패하면
     * (서킷 OPEN, 타임아웃 등) 대신 호출됩니다.
     */
    override fun getProduct(productId: Long): ProductResponse {
        log.warn("Fallback for getProduct(productId={}) triggered.", productId)
        
        // 여기에 대체 로직을 구현합니다.
        // 1. 미리 정의된 기본(Default) 응답 반환
        return ProductResponse(
            id = productId,
            name = "상품 정보 조회 불가", // 사용자에게 표시될 메시지
            price = -1L, // 에러를 나타내는 값
            stockQuantity = 0
        )
        
        // 2. (더 발전된 방식) Redis 같은 캐시에서 오래된(Stale) 데이터라도 조회하여 반환
        // val cachedProduct = redisCache.get("product:$productId")
        // return cachedProduct ?: defaultResponse
    }
}
```

#### 2\. `@FeignClient`에 Fallback 클래스 등록

이제 기존 `ProductServiceClient` 인터페이스의 `@FeignClient` 어노테이션에 `fallback` 속성을 추가하여, 방금 만든 클래스를 지정해 줍니다.

```kotlin
// order-service/src/main/kotlin/com/ecommerce/order/client/ProductServiceClient.kt
package com.ecommerce.order.client

import org.springframework.cloud.openfeign.FeignClient
// ...

@FeignClient(
    name = "product-service",
    fallback = ProductServiceClientFallback::class // 1. Fallback 클래스 등록
)
interface ProductServiceClient {
    // ...
}
```

  * **주의:** Fallback 클래스는 반드시 스프링의 Bean(`@Component`)으로 등록되어 있어야 합니다.

-----

### 최종 동작 시나리오

1.  `order-service`가 `productServiceClient.getProduct(1L)`를 호출합니다.
2.  `product-service`에 장애가 발생하여 서킷 브레이커가 `OPEN`됩니다.
3.  Resilience4j는 더 이상 `CallNotPermittedException`을 던지는 대신, `@FeignClient`에 등록된 \*\*`ProductServiceClientFallback`\*\*의 `getProduct(1L)` 메서드를 호출합니다.
4.  `OrderService`는 예외를 받는 대신, `name`이 "상품 정보 조회 불가"인 `ProductResponse` 객체를 정상적으로 응답받습니다.
5.  `OrderService`는 이 응답을 보고 "아, 지금 상품 정보를 가져올 수 없구나"라고 인지한 뒤, "일시적인 오류로 주문할 수 없습니다. 잠시 후 다시 시도해주세요"와 같은 부드러운 메시지를 사용자에게 보여줄 수 있습니다.

**결과:**
`product-service`의 장애가 발생했음에도 불구하고, `order-service`는 **예외로 인해 비정상 종료되지 않았습니다.** 시스템은 완벽하지는 않지만, 부분적으로나마 **' degraded but still functional'** (기능이 저하되었지만 여전히 동작하는) 상태를 유지하며 안정성을 확보했습니다.

이것으로 동기 통신의 위험을 관리하는 06장을 마칩니다. 우리는 `OpenFeign`으로 편리하게 통신하고, `서킷 브레이커`로 장애를 격리하며, `폴백`으로 우아하게 실패하는 방법을 배웠습니다. 하지만 모든 통신이 동기식일 필요는 없습니다. 다음 07장에서는 시스템 간의 결합도를 더욱 낮추는 '비동기 이벤트 기반 아키텍처'의 세계로 떠나보겠습니다.