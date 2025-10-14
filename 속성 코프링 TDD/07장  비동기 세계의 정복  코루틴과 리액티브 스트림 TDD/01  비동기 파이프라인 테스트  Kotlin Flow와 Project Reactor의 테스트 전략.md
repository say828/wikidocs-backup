## 01\. 비동기 파이프라인 테스트: Kotlin Flow와 Project Reactor의 테스트 전략

`suspend` 함수가 미래의 '단일 값'을 다룬다면, \*\*비동기 스트림(Asynchronous Stream)\*\*은 미래에 걸쳐 도착하는 '여러 값의 시퀀스'를 다룬다. 실시간 주식 시세, 사용자의 마우스 클릭 이벤트, 데이터베이스로부터 스트리밍되는 대용량 데이터 등이 모두 비동기 스트림의 예다.

코틀린 코루틴 세계에서는 **Flow**가, 스프링 리액티브(WebFlux) 생태계에서는 **Project Reactor**의 `Flux`가 이 비동기 스트림을 대표한다. 이러한 데이터 파이프라인을 테스트하는 것은 단순히 최종 결과만 확인하는 것을 넘어, 시간의 흐름에 따라 발생하는 각 이벤트를 정확히 검증해야 하는 새로운 차원의 과제다.

### **Kotlin Flow 테스트 전략**

`kotlinx-coroutines-test` 라이브러리는 Flow를 테스트하는 기본적인 기능을 제공하며, 더 복잡한 시나리오를 위해 **Turbine**이라는 매우 인기 있는 서드파티 라이브러리를 함께 사용하는 것이 업계 표준으로 자리 잡고 있다.

#### **1. 기본 전략: `toList()`로 한번에 모아 검증하기**

테스트하려는 Flow가 유한한 개수의 아이템을 방출(emit)하고 정상적으로 '완료(complete)'되는 것이 확실한 경우, 가장 간단한 방법은 `toList()` 확장 함수를 사용하는 것이다.

**예시: 모든 상품 목록을 Flow로 반환하는 리포지토리**

```kotlin
class ProductFlowRepository {
    private val products = listOf(
        Product(1, "A"), Product(2, "B"), Product(3, "C")
    )

    fun findAll(): Flow<Product> = flow {
        for (p in products) {
            delay(100) // DB I/O 시뮬레이션
            emit(p)
        }
    }
}

// 테스트 코드
class ProductFlowRepositoryTest {
    @Test
    fun `findAll은 모든 상품을 순서대로 방출해야 한다`() = runTest {
        // given
        val repository = ProductFlowRepository()

        // when: Flow의 모든 결과를 List로 수집
        val productList = repository.findAll().toList()

        // then: 수집된 List의 내용과 순서를 검증
        productList.size shouldBe 3
        productList.map { it.name } shouldContainExactly listOf("A", "B", "C")
    }
}
```

`runTest`가 `delay`를 가상 시간으로 처리해주므로, 이 테스트는 지연 시간과 관계없이 즉시 완료된다.

#### **2. 전문가의 전략: Turbine으로 이벤트 하나하나 검증하기**

`toList()`는 Flow가 완료되지 않거나, 아이템이 방출되는 중간 과정을 정밀하게 검증해야 하거나, 에러 발생 상황을 테스트해야 할 때는 한계가 있다. **Turbine**은 이러한 복잡한 시나리오를 위해 탄생한 Flow 테스트 전용 라이브러리다.

`flow.test { ... }` 블록 안에서 우리는 시간의 흐름에 따라 발생하는 이벤트를 하나씩 명시적으로 검증할 수 있다.

**예시: 실시간 시세를 방출하다가 에러를 내는 Ticker**

```kotlin
class PriceTicker {
    fun prices(): Flow<BigDecimal> = flow {
        emit(BigDecimal("100.0"))
        delay(1000)
        emit(BigDecimal("102.5"))
        delay(1000)
        throw IOException("네트워크 연결 끊김")
    }
}

// Turbine을 사용한 테스트 코드 (build.gradle.kts에 'app.cash.turbine:turbine' 의존성 추가 필요)
class PriceTickerTest {
    @Test
    fun `시세를 방출하다가 네트워크 에러가 발생해야 한다`() = runTest {
        // given
        val ticker = PriceTicker()

        // when & then: Turbine의 test 블록으로 Flow를 구독
        ticker.prices().test {
            // 첫 번째 아이템 검증
            awaitItem() shouldBe BigDecimal("100.0")

            // 두 번째 아이템 검증
            awaitItem() shouldBe BigDecimal("102.5")

            // Flow가 완료되지 않고, IOException 에러와 함께 종료되는지 검증
            awaitError() shouldBeInstanceOf IOException::class
        }
    }
}
```

Turbine은 `awaitItem()`, `awaitComplete()`, `awaitError()`, `expectNoEvents()` 등 시간의 흐름에 따른 기대를 명확하게 표현할 수 있는 풍부한 API를 제공하여, 복잡한 비동기 파이프라인도 완벽하게 제어하며 테스트할 수 있게 해준다.

### **Project Reactor (`Flux`) 테스트 전략**

스프링 WebFlux를 사용한다면 `Flux`를 테스트해야 한다. Project Reactor는 \*\*`StepVerifier`\*\*라는 강력한 테스트 도구를 내장하고 있다. `StepVerifier`는 Turbine과 철학적으로 매우 유사하다.

**예시: `Flux`를 반환하는 리액티브 리포지토리**

```kotlin
// 리액티브 스프링 데이터 리포지토리라고 가정
interface ProductReactiveRepository {
    fun findAll(): Flux<Product>
}

// StepVerifier를 사용한 테스트 코드
class ProductReactiveRepositoryTest {
    @Test
    fun `findAll은 모든 상품을 순서대로 방출해야 한다`() {
        // given
        val repository = mockk<ProductReactiveRepository>()
        val flux = Flux.just(Product(1, "A"), Product(2, "B"))
        every { repository.findAll() } returns flux

        // when & then
        StepVerifier.create(repository.findAll())
            .expectNextMatches { it.name == "A" } // 첫 번째 아이템 검증
            .expectNextMatches { it.name == "B" } // 두 번째 아이템 검증
            .verifyComplete() // 스트림이 정상적으로 완료되는지 검증
    }
}
```

`StepVerifier`는 `expectNext`, `expectNextCount`, `expectError`, `verifyComplete` 등 시간 순서에 따라 발생하는 이벤트를 검증하는 체계적인 방법을 제공한다.

비동기 스트림 테스트는 단순히 최종 결과가 아닌, '과정'을 검증하는 것이다. Kotlin Flow에는 Turbine, Project Reactor에는 `StepVerifier`라는 전문가의 도구를 사용하여, 우리는 복잡한 데이터 파이프라인의 모든 움직임을 TDD를 통해 견고하게 설계하고 검증할 수 있다.