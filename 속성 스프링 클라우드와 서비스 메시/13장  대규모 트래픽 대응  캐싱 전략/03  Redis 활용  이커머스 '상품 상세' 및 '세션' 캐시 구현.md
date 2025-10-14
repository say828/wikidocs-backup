## Redis 활용: 이커머스 '상품 상세' 및 '세션' 캐시 구현

이론을 넘어, 이제 실제 이커머스 애플리케이션의 두 가지 핵심 시나리오에 분산 캐시 `Redis`를 적용해 보겠습니다. 바로 \*\*데이터 캐시(Data Cache)\*\*와 \*\*세션 캐시(Session Cache)\*\*입니다.

-----

### 1\. '상품 상세' 캐시 (Data Cache) 심화

13장 02절에서 우리는 `@Cacheable`을 이용해 '상품 상세 정보'를 캐싱하는 기본을 배웠습니다. 이제 TTL(Time-To-Live, 캐시 유효 시간)을 설정하고, 캐시 데이터를 JSON 형태로 저장하는 등 실용적인 설정을 추가해 보겠습니다.

#### JSON 직렬화 및 캐시별 TTL 설정

스프링 캐시의 기본 직렬화 방식은 Java 직렬화입니다. 이는 성능이 좋지 않고, 다른 언어와 호환되지 않으므로, **JSON 형태로 저장**하는 것이 현대적인 방식입니다. `RedisCacheManager` Bean을 직접 설정하여 직렬화 방식과 캐시별 TTL을 커스텀할 수 있습니다.

```kotlin
// product-service/src/main/kotlin/com/ecommerce/product/config/CacheConfig.kt
package com.ecommerce.product.config

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.jsontype.BasicPolymorphicTypeValidator
import com.fasterxml.jackson.module.kotlin.kotlinModule
import org.springframework.cache.annotation.EnableCaching
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.data.redis.cache.RedisCacheConfiguration
import org.springframework.data.redis.cache.RedisCacheManager
import org.springframework.data.redis.connection.RedisConnectionFactory
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer
import org.springframework.data.redis.serializer.RedisSerializationContext
import org.springframework.data.redis.serializer.StringRedisSerializer
import java.time.Duration

@Configuration
@EnableCaching
class CacheConfig {

    @Bean
    fun cacheManager(redisConnectionFactory: RedisConnectionFactory): RedisCacheManager {
        // 1. Jackson ObjectMapper 설정 (Polymorphic Type 지원)
        val objectMapper = ObjectMapper().apply {
            registerModule(kotlinModule())
            activateDefaultTyping(
                BasicPolymorphicTypeValidator.builder().allowIfBaseType(Any::class.java).build(),
                ObjectMapper.DefaultTyping.EVERYTHING
            )
        }
        
        // 2. 기본 캐시 설정: JSON 직렬화, TTL 5분
        val defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(GenericJackson2JsonRedisSerializer(objectMapper)))
            .entryTtl(Duration.ofMinutes(5))

        // 3. 캐시 이름("products")별로 TTL을 다르게 설정
        val customConfigs = mapOf(
            "products" to defaultConfig.entryTtl(Duration.ofMinutes(10)), // 상품 캐시는 10분
            "hot-items" to defaultConfig.entryTtl(Duration.ofMinutes(1))  // 인기 상품 목록은 1분
        )

        return RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(customConfigs)
            .build()
    }
}
```

이제 `product-service`의 `@Cacheable(value = ["products"], ...)` 어노테이션은 자동으로 **10분의 유효 시간**을 가지며, Redis에는 읽기 쉬운 JSON 형태로 데이터가 저장됩니다.

-----

### 2\. '세션' 캐시 (Session Cache)

MSA 환경에서 사용자의 '상태'(예: 로그인 정보, 장바구니)를 관리하는 것은 매우 까다로운 문제입니다. `gateway-service`의 1번 인스턴스에서 로그인했는데, 다음 요청이 2번 인스턴스로 라우팅되면 로그인 정보가 사라지는 문제가 발생합니다.

이 문제를 해결하기 위해, 모든 서비스 인스턴스가 공유하는 **중앙 저장소에 세션을 저장**해야 하며, **Redis**는 이를 위한 완벽한 솔루션입니다.

**Spring Session** 프로젝트는 이 과정을 놀랍도록 간단하게 만들어 줍니다.

#### `gateway-service`에 Spring Session Redis 적용하기

사용자의 모든 요청이 가장 먼저 도착하는 `gateway-service`에 세션 관리 기능을 추가해 봅시다.

**1. 의존성 추가:**

```kotlin
// gateway-service/build.gradle.kts
dependencies {
    // 1. Spring Session과 Redis 연동을 위한 스타터
    implementation("org.springframework.session:spring-session-data-redis")
    // 2. Redis 클라이언트 (spring-boot-starter-data-redis)도 필요
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    // ...
}
```

**2. `@EnableRedisHttpSession` 활성화:**

```kotlin
// gateway-service/src/main/kotlin/com/ecommerce/gateway/config/SessionConfig.kt
package com.ecommerce.gateway.config

import org.springframework.context.annotation.Configuration
import org.springframework.session.data.redis.config.annotation.web.server.EnableRedisWebSession

@Configuration
@EnableRedisWebSession // 1. WebFlux 환경에서는 @EnableRedisWebSession 사용
class SessionConfig
```

> **Note:** Spring WebFlux 기반의 Spring Cloud Gateway에서는 `@EnableRedisHttpSession` 대신 `@EnableRedisWebSession`을 사용합니다.

**3. `application.yml` 설정:**

```yaml
# gateway-service/src/main/resources/application.yml
spring:
  session:
    # 1. 세션 저장소로 redis 지정
    store-type: redis
    # 2. 세션 타임아웃 (30분)
    timeout: 30m
  data:
    redis:
      host: localhost
      port: 6379
```

**이것으로 설정은 끝입니다\!**

이제부터 개발자는 `HttpSession`을 사용하는 기존 코드(예: `session.setAttribute(...)`)를 전혀 수정할 필요가 없습니다. Spring Session이 내부적으로 `HttpSession`의 구현체를 Redis에 데이터를 저장하고 조회하는 구현체로 '자동 교체'해 줍니다.

**동작 방식:**

1.  사용자가 로그인하거나 장바구니에 상품을 담습니다.
2.  `gateway-service`의 Spring Session은 세션 데이터를 생성하여 **Redis에 저장**합니다.
3.  클라이언트(브라우저)에게는 세션 ID가 담긴 `SESSION` 쿠키를 발급합니다.
4.  클라이언트는 이후 모든 요청에 이 `SESSION` 쿠키를 담아 보냅니다.
5.  요청을 받은 게이트웨이 인스턴스(1번이든, 2번이든 상관없이)는 쿠키의 세션 ID를 사용하여 **Redis에서 세션 정보를 조회**하여 사용합니다.

이제 우리의 MSA는 여러 인스턴스로 확장되더라도, 사용자의 로그인 상태나 장바구니 정보를 **상태 손실 없이(stateless-ly)** 일관되게 유지할 수 있게 되었습니다.

다음 절에서는 이 강력한 캐시 시스템의 가장 어려운 숙제, 즉 원본 DB 데이터가 변경되었을 때 캐시를 어떻게 '무효화'하고 일관성을 유지할 것인지에 대한 전략을 다뤄보겠습니다.