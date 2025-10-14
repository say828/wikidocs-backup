## 캐시 무효화(Cache Invalidation) 전략: 이벤트 기반(Kafka) 무효화

Cache-Aside 패턴은 훌륭하지만, 여전히 두 가지 해결되지 않은 문제가 남아있습니다.

1.  **경쟁 상태 (Race Condition):** DB에 쓰기 작업이 일어난 직후, 그리고 캐시가 삭제되기 직전의 아주 짧은 시간 동안, 다른 스레드가 DB에서 **예전 데이터**를 읽어와 캐시에 다시 써버릴 수 있습니다. 그러면 캐시는 영원히 오염된(Stale) 상태로 남게 됩니다.
2.  **외부 시스템의 변경:** 만약 개발자가 운영상의 이유로 DB 클라이언트에 직접 접속하여 상품 가격을 수정했다면 어떻게 될까요? 애플리케이션(`product-service`)은 이 변경을 전혀 알지 못하므로, 캐시를 무효화(삭제)하는 로직(`@CacheEvict`)은 호출되지 않습니다. 캐시는 계속해서 잘못된 데이터를 사용자에게 보여줄 것입니다.

이러한 문제들은 캐시 무효화(Invalidation) 로직을 **쓰기 작업을 수행하는 애플리케이션에 의존**하기 때문에 발생합니다.

-----

### 해답: DB를 '진실의 원천'으로 삼는 이벤트 기반 무효화

이 문제를 해결하는 가장 확실하고 현대적인 방법은, **데이터베이스 자체의 변경을 '이벤트'로 간주**하고, 이 이벤트를 기반으로 캐시를 무효화하는 것입니다.

이는 11장에서 데이터 동기화를 위해 사용했던 \*\*CDC (Change Data Capture)\*\*와 **Debezium**, **Kafka** 파이프라인을 그대로 재활용하는, 매우 우아한 해결책입니다.

**동작 흐름:**

1.  **[DB 변경 발생]** 어떤 주체(애플리케이션, DBA, 배치 작업 등)가 `product-service`의 \*\*쓰기 DB(PostgreSQL)\*\*에 있는 `products` 테이블의 데이터를 변경하고 `COMMIT`합니다.

2.  **[Debezium CDC]** Debezium 커넥터가 이 변경 사항을 데이터베이스의 트랜잭션 로그에서 **실시간으로 캡처**합니다.

3.  **[Kafka 이벤트 발행]** Debezium은 변경 내역(예: `productId: 123`의 `price`가 변경됨)을 담은 이벤트를 `db.product-service.products`와 같은 **Kafka 토픽**으로 발행합니다.

4.  **[이벤트 구독]** **`product-service` 자신**이 이 Kafka 토픽을 구독하는 컨슈머를 가집니다.

5.  **[캐시 무효화]** `product-service`의 컨슈머는 이벤트를 받고, 이벤트 페이로드에서 변경된 데이터의 `ID`(`productId: 123`)를 추출합니다. 그리고 Spring의 `CacheManager`를 사용하여 \*\*프로그래밍 방식으로 Redis에 있는 해당 캐시 키(`products::123`)를 명시적으로 삭제(evict)\*\*합니다.

-----

### 캐시 무효화 컨슈머 구현

`product-service` 내부에, 자신의 DB 변경 이벤트를 구독하여 캐시를 무효화하는 Kafka 컨슈머를 작성합니다.

```kotlin
// product-service/src/main/kotlin/com/ecommerce/product/consumer/CacheInvalidationConsumer.kt
package com.ecommerce.product.consumer

import com.fasterxml.jackson.databind.ObjectMapper
import org.slf4j.LoggerFactory
import org.springframework.cache.CacheManager
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.stereotype.Component

@Component
class CacheInvalidationConsumer(
    private val cacheManager: CacheManager, // 1. Spring CacheManager 주입
    private val objectMapper: ObjectMapper,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @KafkaListener(
        topics = ["db.product-service.products"], // 2. Debezium이 발행하는 토픽 구독
        groupId = "product-cache-invalidator"
    )
    fun handleProductChange(payload: String) {
        try {
            // 3. Debezium 이벤트 페이로드 파싱 (간략화된 예시)
            val event = objectMapper.readValue(payload, DebeziumEvent::class.java)
            val productId = event.payload.after?.id ?: event.payload.before?.id ?: return
            
            // 4. 'products' 캐시 그룹에서 해당 productId 키를 가진 캐시를 무효화
            cacheManager.getCache("products")?.evict(productId)
            
            log.info("Cache invalidated for product ID: {}", productId)
        } catch (e: Exception) {
            log.error("Error while invalidating product cache", e)
        }
    }
}

// Debezium 이벤트 구조를 받기 위한 단순화된 DTO
data class DebeziumEvent(val payload: Payload)
data class Payload(val after: ProductChange?, val before: ProductChange?)
data class ProductChange(val id: Long)
```

### 이벤트 기반 무효화의 장점

  * **신뢰성과 일관성:** 데이터의 유일한 '진실의 원천(Source of Truth)'인 **데이터베이스의 변경**에 직접 반응하므로, 데이터 불일치가 발생할 확률이 극히 낮습니다.
  * **완벽한 디커플링:** 캐시 무효화 로직이 비즈니스 로직(`ProductService`)에서 완전히 분리됩니다. `ProductService`는 이제 캐시에 대해 전혀 신경 쓸 필요 없이 자신의 DB 작업에만 집중하면 됩니다. `@CacheEvict` 같은 어노테이션도 필요 없어집니다.
  * **외부 변경 대응:** DBA가 직접 DB를 수정하는 등 애플리케이션 외부에서 발생한 변경까지 모두 감지하여 캐시를 무효화할 수 있어 매우 견고합니다.

이것으로 13장을 마칩니다. 우리는 로컬 캐시와 분산 캐시의 차이점을 배우고, `Spring Cache` 추상화를 통해 Redis를 연동했으며, 최종적으로는 **이벤트 기반 아키텍처**를 활용하여 MSA 환경에서 발생할 수 있는 복잡한 **캐시 일관성 문제**를 해결하는 고급 전략까지 구현했습니다.

이제 우리의 시스템은 빠르고 안정적입니다. 다음 14장에서는, 이 복잡한 MSA 시스템이 정말로 올바르게 동작하는지 검증하고, 향후 변경에 대한 안정성을 보장하기 위한 **테스트 전략**에 대해 알아보겠습니다.