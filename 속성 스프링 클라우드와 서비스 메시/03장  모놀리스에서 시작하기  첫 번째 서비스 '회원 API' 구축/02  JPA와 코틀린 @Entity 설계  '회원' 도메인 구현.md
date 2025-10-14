## JPA와 코틀린 @Entity 설계: '회원' 도메인 구현

이제 03장 01절에서 만든 API '껍데기'에 '영혼'을 불어넣을 차례입니다. `MemberService`의 `// TODO` 주석을 실제 JPA 코드로 구현하여, 데이터베이스(H2 In-memory DB 또는 실제 RDBMS)와 연동합니다.

-----

### 복습: 코틀린 `@Entity` 설계 원칙 (01장 02절)

우리는 01장 02절에서 JPA 엔티티 설계의 함정을 이미 학습했습니다.

1.  `@Entity`에는 `data class`를 사용하지 않는다. (JPA의 프록시 및 Lazy Loading 메커니즘과 충돌)
2.  `kotlin("plugin.jpa")` 플러그인을 사용한다. (JPA 스펙이 요구하는 `no-arg` 기본 생성자를 자동으로 추가해 줌)
3.  `@MappedSuperclass` 등을 활용하여 공통 속성을 분리한다.

-----

### 1\. (Best Practice) 공통 `BaseEntity` 작성

모든 엔티티는 `createdAt` (생성일시), `updatedAt` (수정일시)을 가져야 합니다. 이를 `@MappedSuperclass`로 분리하여 코드 중복을 제거합니다.

먼저, `MemberApplication`에 JPA Auditing을 활성화합니다.

```kotlin
// /src/main/kotlin/com/ecommerce/member/MemberApplication.kt
package com.ecommerce.member

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.data.jpa.repository.config.EnableJpaAuditing

@EnableJpaAuditing // 1. JPA Auditing 기능 활성화
@SpringBootApplication
class MemberApplication

fun main(args: Array<String>) {
    runApplication<MemberApplication>(*args)
}
```

이제 `BaseEntity`를 작성합니다.

```kotlin
// /src/main/kotlin/com/ecommerce/member/domain/BaseEntity.kt
package com.ecommerce.member.domain

import jakarta.persistence.Column
import jakarta.persistence.EntityListeners
import jakarta.persistence.MappedSuperclass
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.LastModifiedDate
import org.springframework.data.jpa.domain.support.AuditingEntityListener
import java.time.LocalDateTime

@MappedSuperclass // 1. 이 클래스가 공통 매핑 정보를 포함함을 명시
@EntityListeners(AuditingEntityListener::class) // 2. 엔티티 이벤트를 감지하여 Auditing 처리
abstract class BaseEntity {
    
    @CreatedDate // 3. 엔티티 생성 시 자동으로 날짜 주입
    @Column(updatable = false)
    var createdAt: LocalDateTime? = null
        private set // 4. 외부에서 수정 불가능하도록 private set

    @LastModifiedDate // 5. 엔티티 수정 시 자동으로 날짜 주입
    var updatedAt: LocalDateTime? = null
        private set
}
```

-----

### 2\. `Member` @Entity 설계

`BaseEntity`를 상속받아 `Member` 엔티티를 완성합니다.

```kotlin
// /src/main/kotlin/com/ecommerce/member/domain/Member.kt
package com.ecommerce.member.domain

import jakarta.persistence.*

@Entity
@Table(name = "members") // 'member'는 DB 예약어일 수 있으므로 'members' 사용 권장
class Member(
    // --- 1. '생성' 시점에 필요한 핵심 비즈니스 필드 ---
    @Column(nullable = false, unique = true)
    var email: String, // email은 변경 가능성이 있으므로 var

    @Column(nullable = false)
    var name: String, // 이름도 개명 등으로 변경 가능성 있음

    @Column(nullable = false)
    var address: String,

    // (패스워드, 회원 등급 등은 추후 다른 장에서 확장)

) : BaseEntity() { // 2. 공통 속성 상속
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null // 3. id는 DB가 생성하므로 주 생성자에서 제외
}
```

  * **참고:** `kotlin-jpa` 플러그인 덕분에, JPA가 요구하는 `protected` 기본 생성자를 우리가 직접 작성할 필요가 없습니다. 컴파일 시점에 자동으로 생성됩니다.

-----

### 3\. `MemberRepository` (데이터 접근 계층) 정의

`Spring Data JPA`의 `JpaRepository`를 상속받아 데이터 접근 인터페이스를 정의합니다.

```kotlin
// /src/main/kotlin/com/ecommerce/member/domain/MemberRepository.kt
package com.ecommerce.member.domain

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.stereotype.Repository

@Repository
interface MemberRepository : JpaRepository<Member, Long> {
    
    /**
     * Spring Data JPA 쿼리 메서드
     * 이메일을 기준으로 회원을 조회합니다.
     */
    fun findByEmail(email: String): Member?
}
```

-----

### 4\. `MemberService` (비즈니스 로직) 완성

이제 03장 01절의 `MemberService`에 `MemberRepository`를 주입하고 `// TODO`로 표시했던 로직을 실제로 구현합니다.

먼저, 01장 03절에서 배운 '확장 함수'를 이용해 DTO와 Entity 간의 변환 로직을 분리합니다.

```kotlin
// /src/main/kotlin/com/ecommerce/member/api/MemberDto.kt (파일에 확장 함수 추가)
package com.ecommerce.member.api

import com.ecommerce.member.domain.Member

// ... (JoinMemberRequest, MemberResponse data class 정의) ...

/**
 * 01장 03절에서 배운 '확장 함수'
 * DTO -> Entity 변환
 */
fun JoinMemberRequest.toEntity(): Member {
    return Member(
        email = this.email,
        name = this.name,
        address = this.address
    )
}

/**
 * Entity -> DTO 변환
 */
fun Member.toResponse(): MemberResponse {
    // Entity의 id는 null일 수 없다고 확신 (DB에서 조회했거나 저장된 후이므로)
    return MemberResponse(
        id = this.id!!,
        email = this.email,
        name = this.name
    )
}
```

이제 `MemberService`를 완성합니다.

```kotlin
// /src/main/kotlin/com/ecommerce/member/service/MemberService.kt (수정)
package com.ecommerce.member.service

import com.ecommerce.member.api.JoinMemberRequest
import com.ecommerce.member.api.MemberResponse
import com.ecommerce.member.api.toEntity // 1. 확장 함수 Import
import com.ecommerce.member.api.toResponse // 1. 확장 함수 Import
import com.ecommerce.member.domain.MemberRepository
import jakarta.persistence.EntityNotFoundException
import org.springframework.data.repository.findByIdOrNull // 2. 코틀린 JpaRepository 확장
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional(readOnly = true)
class MemberService(
    private val memberRepository: MemberRepository // 3. Repository 주입
) {
    
    @Transactional // 4. 쓰기 트랜잭션 오버라이드
    fun join(request: JoinMemberRequest): MemberResponse {
        // 5. 이메일 중복 검사
        if (memberRepository.findByEmail(request.email) != null) {
            throw IllegalArgumentException("이미 가입된 이메일입니다: ${request.email}")
        }
        
        // 6. DTO -> Entity 변환 (확장 함수 사용)
        val newMember = request.toEntity()
        
        // 7. DB에 저장 (영속화)
        val savedMember = memberRepository.save(newMember)
        
        // 8. Entity -> DTO 변환 후 반환
        return savedMember.toResponse()
    }

    fun getMember(memberId: Long): MemberResponse {
        // 9. ID로 회원 조회 (코틀린 확장인 findByIdOrNull 사용)
        val member = memberRepository.findByIdOrNull(memberId)
            ?: throw EntityNotFoundException("해당 ID의 회원을 찾을 수 없습니다: $memberId")
            
        // 10. Entity -> DTO 변환
        return member.toResponse()
    }
}
```

-----

이것으로 `회원 API`의 모든 계층(Controller-Service-Repository)이 연결되어 **완전한 기능**을 갖추게 되었습니다. 우리는 코틀린의 장점(data class, 확장 함수, null-safety)과 스프링 부트 JPA의 강력함을 결합하여 깨끗하고 견고한 CRUD API를 완성했습니다.

다음 절에서는, 우리가 작성한 이 코드가 '제대로' 동작하는지 **테스트 코드를 작성**하여 검증할 것입니다.