## R2DBC: 코루틴을 이용한 논블로킹 DB 액세스

12장 03절까지 우리는 `WebFlux`와 `코루틴`을 사용하여 애플리케이션의 '입구'(Controller)와 '중간 처리'(Service)를 논블로킹으로 만들었습니다. 하지만 결정적인 병목이 하나 남아있습니다. 바로 **데이터베이스 접근**입니다.

우리가 지금까지 사용해 온 `Spring Data JPA`는 내부적으로 **JDBC(Java Database Connectivity)** API를 사용합니다. JDBC는 근본적으로 **블로킹(Blocking)** 방식으로 동작하도록 설계되었습니다. `suspend` 함수 안에서 JPA 리포지토리 메서드를 호출해봐야, DB 커넥션을 얻고 쿼리를 실행하는 과정에서 **스레드는 어김없이 블로킹**됩니다. 이는 WebFlux와 코루틴으로 얻은 논블로킹의 모든 이점을 수포로 돌리는 행위입니다.

진정한 엔드-투-엔드(End-to-End) 논블로킹을 달성하기 위해서는, 데이터베이스 드라이버 레벨부터 리액티브해야 합니다. 그 해답이 바로 **R2DBC**입니다.

-----

### R2DBC (Reactive Relational Database Connectivity)란?

R2DBC는 관계형 데이터베이스를 위한 **리액티브 API 명세**입니다. JDBC의 리액티브 버전이라고 생각하면 쉽습니다. R2DBC 드라이버는 DB와의 모든 통신을 논블로킹 I/O 방식으로 처리하며, 쿼리 결과 또한 `Mono`나 `Flux` 같은 리액티브 스트림(Publisher)으로 반환합니다.

그리고 **Spring Data R2DBC**는 이 R2DBC 위에 구축된, 우리가 익숙한 `Repository` 패턴을 제공하는 프레임워크입니다. 가장 환상적인 점은, Spring Data R2DBC가 **코틀린 코루틴을 완벽하게 지원**한다는 것입니다.

-----

### `member-service`를 R2DBC로 전환하기

#### 1\. 의존성 변경 (`build.gradle.kts`)

JPA 관련 의존성을 제거하고, R2DBC 관련 의존성을 추가합니다.

```kotlin
// member-service/build.gradle.kts
dependencies {
    // 1. 기존 JPA 스타터 제거
    // implementation("org.springframework.boot:spring-boot-starter-data-jpa")

    // 2. R2DBC 스타터 추가
    implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")

    // 3. 사용할 DB에 맞는 R2DBC 드라이버 추가 (H2 예시)
    implementation("io.r2dbc:r2dbc-h2")
    // (PostgreSQL 사용 시: implementation("org.postgresql:r2dbc-postgresql"))
    
    // ... (webflux, coroutines-reactor 등) ...
}
```

#### 2\. `application.yml` 설정 변경

DB 접속 정보를 R2DBC용으로 변경합니다. URL의 스킴이 `jdbc:`에서 `r2dbc:`로 바뀝니다.

```yaml
# member-service/src/main/resources/application.yml
spring:
  # 1. JPA 설정 삭제 및 R2DBC 설정 추가
  r2dbc:
    url: r2dbc:h2:mem:///testdb;DB_CLOSE_DELAY=-1;
    username: sa
    password:
```

#### 3\. 엔티티 재정의 (JPA 어노테이션 제거)

R2DBC는 JPA 표준이 아니므로, `jakarta.persistence` 어노테이션(`@Entity`, `@GeneratedValue` 등)을 사용하지 않습니다. 대신 Spring Data의 공통 어노테이션(`@Table`, `@Id`)을 사용합니다.

```kotlin
// member-service/src/main/kotlin/com/ecommerce/member/domain/Member.kt
package com.ecommerce.member.domain

import org.springframework.data.annotation.Id
import org.springframework.data.relational.core.mapping.Table

@Table("members") // 1. JPA의 @Entity 대신 @Table 사용
class Member(
    // 2. 주 생성자로 모든 필드를 받도록 설계하는 것이 편리
    @Id
    var id: Long? = null, // ID는 DB가 생성 후 채워주므로 var로 선언

    val email: String,
    val name: String,
    val address: String,
)
```

#### 4\. `CoroutineCrudRepository` 구현

`JpaRepository` 대신, 코루틴을 네이티브로 지원하는 \*\*`CoroutineCrudRepository`\*\*를 상속받아 리포지토리 인터페이스를 정의합니다.

```kotlin
// member-service/src/main/kotlin/com/ecommerce/member/domain/MemberRepository.kt
package com.ecommerce.member.domain

import org.springframework.data.r2dbc.repository.Query
import org.springframework.data.repository.kotlin.CoroutineCrudRepository
import org.springframework.stereotype.Repository

@Repository
// 1. JpaRepository 대신 CoroutineCrudRepository 상속
interface MemberRepository : CoroutineCrudRepository<Member, Long> {

    /**
     * CoroutineCrudRepository는 이미 save, findById, findAll 등을
     * 'suspend' 함수로 기본 제공합니다.
     */

    /**
     * 커스텀 쿼리 메서드 또한 'suspend' 함수로 선언 가능
     */
    @Query("SELECT * FROM members WHERE email = :email")
    suspend fun findByEmail(email: String): Member?
}
```

`CoroutineCrudRepository`를 상속받는 것만으로, 우리는 `save`, `findById` 등 모든 기본 CRUD 메서드를 `suspend` 함수로 사용할 수 있게 됩니다\!

#### 5\. 서비스 계층 완성

12장 03절에서 이미 `suspend` 기반으로 설계했던 `MemberService` 코드는 이제 **거의 수정 없이** 그대로 동작합니다.

```kotlin
// member-service/src/main/kotlin/com/ecommerce/member/service/MemberService.kt

@Service
class MemberService(
    private val memberRepository: MemberRepository // 이제 CoroutineCrudRepository 구현체
) {

    suspend fun join(request: JoinMemberRequest): MemberResponse {
        if (memberRepository.findByEmail(request.email) != null) {
            throw IllegalArgumentException("이미 가입된 이메일입니다.")
        }
        
        // memberRepository.save()는 이제 논블로킹으로 동작하는 suspend 함수
        val newMember = memberRepository.save(request.toEntity())
        return newMember.toResponse()
    }

    suspend fun getMember(memberId: Long): MemberResponse {
        // findById() 또한 논블로킹으로 동작하는 suspend 함수
        val member = memberRepository.findById(memberId)
            ?: throw EntityNotFoundException("...")
        return member.toResponse()
    }
}
```

-----

### R2DBC의 트레이드오프

R2DBC는 강력하지만, 성숙한 JPA/Hibernate에 비해 몇 가지 트레이드오프가 있습니다.

  * **제한적인 기능:** JPA가 제공하는 지연 로딩(Lazy Loading), 복잡한 연관관계 매핑, 캐싱, Dirty Checking 같은 고급 기능 대부분을 지원하지 않습니다.
  * **스키마 자동 생성 부재:** `ddl-auto` 옵션이 없으므로, `Flyway`나 `Liquibase` 같은 별도의 DB 마이그레이션 도구를 사용하여 스키마를 직접 관리해야 합니다.

-----

이것으로 우리의 완전한 논블로킹 스택이 완성되었습니다. 이제 사용자의 HTTP 요청부터 DB 쿼리까지, 애플리케이션의 모든 계층이 코루틴 기반의 논블로킹 I/O로 동작합니다. 이를 통해 우리는 적은 수의 스레드만으로 대규모 동시 트래픽을 효율적으로 처리할 수 있는, 진정한 고성능 마이크로서비스를 구축할 수 있는 기반을 갖추게 되었습니다.