## Testcontainers: 로컬 환경에서 실제 DB/Kafka/Redis 띄우고 통합 테스트하기

단위 테스트는 매우 빠르고 안정적이지만, 명확한 한계를 가집니다. 바로 \*\*'Mock은 실제가 아니다'\*\*라는 점입니다. `MockK`로 만든 가짜 `OrderRepository`는 실제 데이터베이스의 제약 조건(Constraint), SQL 문법 오류, 커넥션 풀 문제 등을 전혀 검증해주지 못합니다.

이 간극을 메우는 것이 바로 \*\*통합 테스트(Integration Test)\*\*입니다. 통합 테스트는 Mock을 사용하는 대신, **실제(Real) 외부 의존성**과 연동하여 서비스가 올바르게 동작하는지 검증합니다. `order-service`의 경우, 실제 PostgreSQL 데이터베이스와 연동하여 `OrderRepository`의 JPQL 쿼리가 정상적으로 실행되는지 테스트해야 합니다.

하지만 이를 위해 개발자 PC마다 PostgreSQL, Kafka, Redis를 직접 설치하고 관리하는 것은 매우 번거롭고, OS에 따라 환경이 달라져 테스트가 깨지는 원인이 됩니다.

-----

### Testcontainers: 테스트를 위한 경량 Docker 환경 🐳

이 문제를 해결하는 가장 현대적이고 강력한 도구가 바로 **Testcontainers**입니다.

**Testcontainers**는 JUnit이나 Kotest 같은 테스트 프레임워크와 연동하여, **테스트 코드 내에서 프로그래밍 방식으로 도커 컨테이너를 생성하고 관리**할 수 있게 해주는 라이브러리입니다.

**동작 방식:**

1.  테스트가 시작되면, Testcontainers는 도커를 이용해 PostgreSQL 컨테이너를 **즉석에서(On-the-fly)** 띄웁니다.
2.  컨테이너가 실행되면, 동적으로 할당된 IP 주소와 포트, DB 이름, 사용자 정보 등을 가져와 스프링 부트의 `application.yml` 설정에 **자동으로 덮어씁니다.**
3.  우리의 테스트 코드는 이 임시 컨테이너를 마치 실제 DB처럼 사용하여 테스트를 수행합니다.
4.  테스트가 끝나면, Testcontainers는 사용했던 컨테이너를 **자동으로 종료하고 삭제**합니다.

이 덕분에 개발자는 자신의 로컬 머신에 아무것도 설치할 필요 없이, 오직 도커만 설치되어 있다면 언제 어디서든 일관된 환경에서 깨끗하게 격리된 통합 테스트를 실행할 수 있습니다.

-----

### 실전 예제: `OrderRepository` 통합 테스트

`order-service`의 `OrderRepository`가 DB와 올바르게 상호작용하는지 Testcontainers를 이용해 테스트해 봅시다.

#### 1\. 의존성 추가

`order-service`의 `build.gradle.kts`에 Testcontainers 관련 의존성을 추가합니다.

```kotlin
// order-service/build.gradle.kts
dependencies {
    // ...
    // 1. Kotest와 Testcontainers를 연동하기 위한 라이브러리
    testImplementation("io.kotest.extensions:kotest-extensions-testcontainers:2.0.2")
    // 2. Testcontainers 핵심 라이브러리
    testImplementation("org.testcontainers:testcontainers:1.19.7")
    // 3. 사용할 DB에 맞는 모듈 추가 (PostgreSQL)
    testImplementation("org.testcontainers:postgresql:1.19.7")
    // 4. Spring Data R2DBC를 테스트하기 위해 필요
    testImplementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
    testImplementation("io.r2dbc:r2dbc-postgresql")
}
```

#### 2\. `Repository` 통합 테스트 코드 작성

`@DataR2dbcTest` 어노테이션을 사용하여 데이터 접근 계층만 테스트하고, `Testcontainers`를 Kotest와 연동합니다.

```kotlin
// order-service/src/test/kotlin/com/ecommerce/order/domain/OrderRepositoryTest.kt
package com.ecommerce.order.domain

import io.kotest.core.spec.style.DescribeSpec
import io.kotest.extensions.testcontainers.perSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.test.runTest
import org.springframework.boot.test.autoconfigure.data.r2dbc.DataR2dbcTest
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer

// 1. @DataR2dbcTest: R2DBC 관련 Bean들만 로드하여 테스트
//    (또는 @SpringBootTest를 사용하여 전체 컨텍스트를 로드할 수도 있음)
@DataR2dbcTest 
class OrderRepositoryTest(
    private val orderRepository: OrderRepository // 2. 실제 Repository 주입
) : DescribeSpec({

    // 3. Companion object에 Testcontainers 설정을 정적으로 정의
    companion object {
        // 4. PostgreSQL 15 버전의 컨테이너를 생성
        val postgresqlContainer = PostgreSQLContainer("postgres:15-alpine")

        /**
         * 5. @DynamicPropertySource: Testcontainers가 띄운 컨테이너의
         * 동적 포트, DB 이름 등을 스프링의 설정 값으로 주입
         */
        @JvmStatic
        @DynamicPropertySource
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.r2dbc.url") { 
                "r2dbc:postgresql://${postgresqlContainer.host}:${postgresqlContainer.firstMappedPort}/${postgresqlContainer.databaseName}" 
            }
            registry.add("spring.r2dbc.username", postgresqlContainer::getUsername)
            registry.add("spring.r2dbc.password", postgresqlContainer::getPassword)
            registry.add("spring.flyway.url", postgresqlContainer::getJdbcUrl) // Flyway 스키마 마이그레이션용
        }
    }

    // 6. Kotest의 perSpec 리스너를 통해 이 테스트 클래스가 실행될 때 컨테이너를 시작/종료
    listener(postgresqlContainer.perSpec())

    // 7. 각 테스트 케이스 실행 전, 데이터를 초기화하여 격리 보장
    beforeEach {
        orderRepository.deleteAll()
    }

    describe("save and findById") {
        it("주문을 저장하고 ID로 조회할 수 있어야 한다") {
            runTest { // 코루틴 테스트를 위한 runTest
                // Given
                val newOrder = Order(memberId = 1L, /* ... */)

                // When
                val savedOrder = orderRepository.save(newOrder)
                val foundOrder = orderRepository.findById(savedOrder.id!!)

                // Then
                foundOrder shouldNotBe null
                foundOrder?.memberId shouldBe 1L
            }
        }
    }
    
    describe("findByMemberId") {
        it("특정 회원의 주문 목록을 조회할 수 있어야 한다") {
            runTest {
                // Given
                orderRepository.save(Order(memberId = 1L, /* ... */))
                orderRepository.save(Order(memberId = 2L, /* ... */))
                orderRepository.save(Order(memberId = 1L, /* ... */))

                // When
                val member1Orders = orderRepository.findByMemberId(1L).toList()

                // Then
                member1Orders.size shouldBe 2
            }
        }
    }
})
```

이제 이 테스트를 실행하면, Testcontainers는 백그라운드에서 PostgreSQL 도커 컨테이너를 실행하고, Flyway가 DB 스키마를 생성한 뒤, `OrderRepository`는 이 실제 DB를 상대로 `save`와 `findById`를 수행하여 JPQL 쿼리의 정확성을 검증합니다.

-----

Testcontainers를 사용하면 DB뿐만 아니라 **Redis, Kafka, Elasticsearch 등 도커로 실행할 수 있는 거의 모든 의존성**을 테스트 코드 내에서 관리할 수 있습니다. 이를 통해 우리는 Mocking만으로는 불가능했던, 실제 환경과 매우 유사한 수준의 신뢰도 높은 통합 테스트를 CI 파이프라인에서 자동화하여 MSA의 안정성을 한 차원 더 높일 수 있습니다.