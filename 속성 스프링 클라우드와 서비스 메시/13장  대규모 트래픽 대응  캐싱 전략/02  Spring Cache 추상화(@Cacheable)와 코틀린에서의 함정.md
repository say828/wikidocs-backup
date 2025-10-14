## Spring Cache 추상화(@Cacheable)와 코틀린에서의 함정

13장 01절에서 우리는 분산 캐시로 Redis를 선택했습니다. 그렇다면 이제 `product-service`의 코드에 `RedisTemplate`을 주입받아 `redisTemplate.opsForValue().get(...)` 같은 Redis 전용 코드를 직접 작성해야 할까요?

그렇게 하면 우리의 비즈니스 로직이 **Redis라는 특정 기술에 강하게 결합**됩니다. 만약 나중에 캐시 전략이 바뀌어 Caffeine이나 다른 캐시 솔루션으로 교체해야 한다면, 모든 서비스 코드를 수정해야 하는 끔찍한 상황이 발생합니다.

이 문제를 해결하기 위해 스프링은 **`Spring Cache` 추상화**를 제공합니다.

-----

### `@Cacheable`: 선언적 캐싱의 마법

Spring Cache 추상화의 핵심은 `JPA`처럼, 개발자가 특정 구현 기술이 아닌 \*\*'어노테이션'\*\*에 맞춰 코드를 작성하게 하는 것입니다. 개발자는 캐싱 로직의 'How(어떻게)'가 아닌 'What(무엇을)'에만 집중하면 됩니다.

가장 대표적인 어노테이션이 바로 \*\*`@Cacheable`\*\*입니다.

`@Cacheable`은 메서드에 붙이는 어노테이션으로, 다음과 같이 동작합니다.

1.  메서드가 호출되면, 스프링은 먼저 캐시를 확인합니다.
2.  **(Cache Hit):** 캐시에 해당 데이터가 있으면, **메서드의 본문(Body)을 아예 실행하지 않고** 캐시된 값을 즉시 반환합니다.
3.  **(Cache Miss):** 캐시에 데이터가 없으면, 원래대로 메서드의 본문을 실행합니다.
4.  메서드가 성공적으로 값을 반환하면, 스프링은 그 반환값을 **캐시에 자동으로 저장**한 뒤, 클라이언트에게 반환합니다.

이 외에도 다음과 같은 어노테이션들이 있습니다.

  * `@CachePut`: 항상 메서드를 실행하고, 그 결과를 캐시에 '갱신'합니다. (데이터 업데이트 시 유용)
  * `@CacheEvict`: 메서드를 실행하고, 캐시에서 데이터를 '삭제'합니다. (데이터 삭제 시 유용)

-----

### `product-service`에 Redis 캐시 적용하기

#### 1\. 의존성 추가

`product-service`의 `build.gradle.kts`에 캐시와 Redis 관련 스타터를 추가합니다.

```kotlin
// product-service/build.gradle.kts
dependencies {
    // 1. 스프링 캐시 추상화 스타터
    implementation("org.springframework.boot:spring-boot-starter-cache")
    // 2. Redis 연동 스타터 (내부에 Lettuce 클라이언트 포함)
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    // ...
}
```

#### 2\. 캐시 활성화 및 Redis 설정

메인 애플리케이션 클래스에 `@EnableCaching`을 추가하여 캐시 기능을 활성화합니다.

```kotlin
// product-service/src/main/kotlin/com/ecommerce/product/ProductApplication.kt
@SpringBootApplication
@EnableCaching // 캐시 기능 활성화
class ProductApplication
// ...
```

`application.yml`에 캐시 타입을 `redis`로 지정하고, Redis 서버 접속 정보를 입력합니다.

```yaml
# product-service/src/main/resources/application.yml
spring:
  cache:
    type: redis # 1. 사용할 캐시 구현체로 redis 지정
  data:
    redis:
      host: localhost
      port: 6379
```

#### 3\. `@Cacheable` 적용

이제 `ProductService`의 상품 조회 메서드에 `@Cacheable` 어노테이션을 붙여주기만 하면 됩니다.

```kotlin
// product-service/src/main/kotlin/com/ecommerce/product/service/ProductService.kt
@Service
class ProductService(
    private val productRepository: ProductRepository
) {
    /**
     * @Cacheable 어노테이션을 통해 이 메서드의 결과를 캐싱
     */
    @Cacheable(
        value = ["products"],     // 1. 캐시의 이름(그룹) 지정
        key = "#productId"        // 2. 캐시 키(Key) 지정 (SpEL 사용)
        // unless = "#result == null" // (선택) 결과가 null이면 캐싱하지 않음
    )
    fun getProduct(productId: Long): ProductResponse {
        log.info("### Finding product from DATABASE... productId: {} ###", productId)
        val product = productRepository.findByIdOrNull(productId)
            ?: throw EntityNotFoundException("상품을 찾을 수 없습니다.")
        return product.toResponse()
    }
}
```

  * **`value = ["products"]`**: 이 캐시 데이터가 저장될 '공간'의 이름입니다. Redis에서는 보통 `products::[key]` 형태의 Key로 저장됩니다.
  * **`key = "#productId"`**: 캐시를 구분하는 고유 키를 생성하는 규칙입니다. \*\*Spring Expression Language(SpEL)\*\*를 사용하며, `#productId`는 이 메서드의 파라미터인 `productId`의 값을 의미합니다. 즉, `getProduct(123)` 호출의 캐시 키는 `123`이 됩니다.

이제 애플리케이션을 실행하고 `getProduct(123)`를 처음 호출하면 "Finding product from DATABASE..." 로그가 찍히지만, **두 번째 호출부터는 해당 로그가 찍히지 않고** 즉시 응답이 오는 것을 확인할 수 있습니다.

-----

### ⚠️ 코틀린에서의 함정: `final`과 AOP 프록시

사실 여기에는 중요한 함정이 숨어있습니다. Spring의 `@Cacheable`, `@Transactional` 같은 어노테이션은 \*\*AOP(관점 지향 프로그래밍)\*\*와 **프록시(Proxy)** 패턴을 기반으로 동작합니다. 스프링은 런타임에 원본 `ProductService`를 감싸는 '프록시 객체'를 만들어, `getProduct` 메서드 호출을 가로채 캐싱 로직을 먼저 수행합니다.

프록시 객체가 원본 객체의 메서드를 가로채려면(Override), 해당 메서드와 클래스가 **상속 가능해야** 합니다. 즉, `open` 키워드가 붙어있어야 합니다. 하지만 **코틀린의 모든 클래스와 메서드는 기본적으로 `final`** 입니다.

만약 아무 설정 없이 위 코드를 작성하면, 프록시가 `getProduct` 메서드를 오버라이드할 수 없어 **`@Cacheable` 어노테이션이 조용히 무시됩니다\!**

**해결책: `kotlin-spring` (all-open) 플러그인**

다행히, 우리는 03장에서 프로젝트를 처음 설정할 때부터 `build.gradle.kts`에 이 문제를 해결해 줄 플러그인을 이미 추가해 두었습니다.

```kotlin
// build.gradle.kts
plugins {
    kotlin("plugin.spring") version "1.9.24"
}
```

이 **`kotlin-spring` (all-open)** 플러그인은, `@Component`, `@Service`, `@Configuration` 등 스프링 관련 어노테이션이 붙은 클래스와 그 안의 모든 메서드를 컴파일 시점에 **자동으로 `open` 상태로 변경**해 줍니다.

결론적으로, 우리가 이 플러그인을 꾸준히 사용해왔기 때문에 `@Cacheable`이 '마법처럼' 동작하는 것입니다. 이 원리를 이해하는 것은 코틀린과 스프링을 함께 사용할 때 매우 중요합니다.