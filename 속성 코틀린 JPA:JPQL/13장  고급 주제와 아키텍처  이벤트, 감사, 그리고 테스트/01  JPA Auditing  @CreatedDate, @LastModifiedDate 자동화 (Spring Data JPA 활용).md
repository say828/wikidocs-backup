## 01\. JPA Auditing: @CreatedDate, @LastModifiedDate 자동화 (Spring Data JPA 활용)

대부분의 애플리케이션에서는 데이터가 언제 생성되고 언제 마지막으로 수정되었는지 기록하는 것이 매우 중요하다. 이를 위해 거의 모든 테이블에 `createdAt`, `updatedAt`과 같은 컬럼을 추가한다. 이전 절에서 배운 엔티티 리스너의 `@PrePersist`, `@PreUpdate`를 사용하면 이 값을 수동으로 채워 넣을 수 있다.

```kotlin
// 수동으로 처리하는 방법 (반복적이고 비효율적)
@PrePersist
fun prePersist() {
    createdAt = LocalDateTime.now()
    updatedAt = LocalDateTime.now()
}

@PreUpdate
fun preUpdate() {
    updatedAt = LocalDateTime.now()
}
```

하지만 모든 엔티티마다 이 코드를 반복해서 작성하는 것은 엄청난 보일러플레이트(boilerplate)다. 스프링 데이터 JPA는 이러한 감사(Auditing) 기능을 완벽하게 자동화할 수 있는 환상적인 방법을 제공한다.

-----

### **Spring Data JPA Auditing 설정 3단계**

**1단계: Auditing 기능 활성화**
먼저, 스프링 부트 애플리케이션에 Auditing 기능을 사용하겠다고 알려주어야 한다. 메인 애플리케이션 클래스에 `@EnableJpaAuditing` 어노테이션을 추가한다.

**Application.kt**

```kotlin
@EnableJpaAuditing // JPA Auditing 기능 활성화!
@SpringBootApplication
class JpaMasterApplication

fun main(args: Array<String>) {
    runApplication<JpaMasterApplication>(*args)
}
```

**2단계: 공통 필드를 위한 MappedSuperclass 생성**
다음으로, 생성/수정 시간 필드를 공통으로 관리할 부모 클래스를 만든다. 이 클래스는 `@MappedSuperclass`로 만들고, 스프링 데이터 JPA가 제공하는 `AuditingEntityListener`를 리스너로 등록해야 한다.

**BaseTimeEntity.kt**

```kotlin
import org.springframework.data.annotation.CreatedDate
import org.springframework.data.annotation.LastModifiedDate
import org.springframework.data.jpa.domain.support.AuditingEntityListener
import jakarta.persistence.EntityListeners
import jakarta.persistence.MappedSuperclass
import java.time.LocalDateTime

@MappedSuperclass
@EntityListeners(AuditingEntityListener::class) // Auditing 리스너 등록
abstract class BaseTimeEntity {

    @CreatedDate // 엔티티 생성 시 시간 자동 저장
    @Column(updatable = false)
    var createdAt: LocalDateTime? = null

    @LastModifiedDate // 엔티티 수정 시 시간 자동 저장
    var updatedAt: LocalDateTime? = null
}
```

  * `@EntityListeners(AuditingEntityListener.class)`: 이 클래스에 Auditing 기능을 적용하라는 의미다. `AuditingEntityListener`가 내부적으로 `@PrePersist`, `@PreUpdate` 로직을 모두 처리해준다.
  * `@CreatedDate`: 최초 저장(`persist`) 시 현재 시간을 자동으로 주입한다.
  * `@LastModifiedDate`: 데이터가 수정될 때마다 현재 시간을 자동으로 주입한다.

**3단계: 엔티티가 공통 클래스를 상속**
마지막으로, Auditing이 필요한 모든 엔티티가 위에서 만든 `BaseTimeEntity`를 상속받도록 수정하면 끝이다.

**Member.kt**

```kotlin
@Entity
class Member(
    // ...
) : BaseTimeEntity() // BaseTimeEntity 상속!
```

이제 모든 준비가 끝났다. `Member` 엔티티를 저장하면 `createdAt`과 `updatedAt`이 자동으로 채워지고, 이후 해당 `Member`를 수정하면 `updatedAt` 필드가 현재 시간으로 자동 갱신된다. 개발자는 더 이상 날짜/시간 관련 코드를 단 한 줄도 작성할 필요가 없다.

-----

### **생성자/수정자 정보 자동화 (`@CreatedBy`, `@LastModifiedBy`)**

누가 데이터를 생성하고 수정했는지 기록하는 것도 가능하다. `@CreatedBy`, `@LastModifiedBy` 어노테이션을 사용하면 된다.

이를 위해서는 현재 로그인한 사용자가 누구인지 스프링 시큐리티 등과 연동하여 알려주는 `AuditorAware` 인터페이스의 구현체를 스프링 빈으로 등록해야 한다.

**AuditorAware 구현체 예시**

```kotlin
@Configuration
class JpaConfig {
    @Bean
    fun auditorProvider(): AuditorAware<String> {
        // TODO: 스프링 시큐리티 컨텍스트에서 실제 사용자 ID를 가져오는 로직 구현
        // 지금은 "admin"을 하드코딩하여 반환
        return AuditorAware { Optional.of("admin") }
    }
}
```

이제 `BaseEntity`에 `@CreatedBy`, `@LastModifiedBy` 필드를 추가하면, 데이터를 생성/수정할 때마다 `auditorProvider`가 반환한 값("admin")이 자동으로 기록된다.

스프링 데이터 JPA의 Auditing 기능은 엔티티 리스너와 `@MappedSuperclass`의 가장 훌륭한 활용 사례다. 반복적인 코드를 제거하고, 애플리케이션의 핵심 관심사와 횡단 관심사를 완벽하게 분리하여 코드의 품질을 극적으로 향상시킨다.

이제 우리는 엔티티의 생명주기에 맞춰 부가 기능을 자동화하는 법을 배웠다. 그렇다면 우리가 작성한 이 복잡한 JPA 코드들이 제대로 동작하는지는 어떻게 보장할 수 있을까? 다음 절에서는 데이터 접근 계층을 테스트하는 방법에 대해 알아본다.