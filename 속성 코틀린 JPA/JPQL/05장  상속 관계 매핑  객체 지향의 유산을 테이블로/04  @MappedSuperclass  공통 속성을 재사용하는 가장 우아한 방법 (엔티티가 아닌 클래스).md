## 04\. @MappedSuperclass: 공통 속성을 재사용하는 가장 우아한 방법 (엔티티가 아닌 클래스)

지금까지 다룬 상속 매핑 전략들은 모두 'is-a' 관계, 즉 "Book is an Item"과 같은 명확한 상속 관계를 전제로 한다. 하지만 실무에서는 상속 관계는 아니지만 여러 엔티티에 걸쳐 공통적으로 사용되는 속성들이 있다. 대표적인 예가 `createdAt`, `createdBy`, `updatedAt`과 같은 등록/수정 관련 메타데이터다. `Member`, `Team`, `Order`, `Product` 등 거의 모든 엔티티가 이 속성들을 필요로 하지만, 이들 사이에 상속 관계가 있는 것은 아니다.

이런 경우, 모든 엔티티 클래스에 해당 필드들을 복사-붙여넣기 하는 것은 매우 비효율적이고 유지보수에도 좋지 않다. 바로 이 문제를 해결하기 위해 JPA는 \*\*`@MappedSuperclass`\*\*라는 강력한 기능을 제공한다.

-----

### **`@MappedSuperclass`란 무엇인가?**

`@MappedSuperclass`는 이름 그대로 \*\*매핑 정보만 제공하는 부모 클래스(Superclass)\*\*를 의미한다. 이 어노테이션이 붙은 클래스는 다음과 같은 특징을 가진다.

  * **엔티티가 아니다**: 이 클래스는 테이블과 직접 매핑되지 않으며, 따라서 데이터베이스에 테이블이 생성되지 않는다.
  * **직접 조회/저장 불가**: `entityManager.find()`나 JPQL에서 이 클래스를 사용할 수 없다.
  * **속성만 상속**: 이 클래스를 상속받는 자식 엔티티들은 부모가 가진 매핑 정보(필드, `@Column` 설정 등)를 그대로 물려받아 자신의 테이블에 컬럼으로 생성한다.

즉, `@MappedSuperclass`는 객체 세상에서의 코드 재사용을 위한 도구일 뿐, 데이터베이스의 상속과는 아무런 관련이 없다.

-----

### **적용 코드 예시**

모든 엔티티가 공통으로 가질 생성 시간(`createdAt`)과 수정 시간(`updatedAt`) 필드를 가진 `BaseEntity`를 만들어 보자.

**BaseEntity.kt (`@MappedSuperclass`)**

```kotlin
import jakarta.persistence.Column
import jakarta.persistence.MappedSuperclass
import java.time.LocalDateTime

@MappedSuperclass // 나는 매핑 정보만 제공하는 부모 클래스야.
abstract class BaseEntity {
    @Column(updatable = false)
    var createdAt: LocalDateTime = LocalDateTime.now()
    var updatedAt: LocalDateTime = LocalDateTime.now()
}
```

**Member.kt (자식 엔티티)**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long = 0,
    var username: String
) : BaseEntity() // BaseEntity를 상속받는다.
```

**Team.kt (자식 엔티티)**

```kotlin
@Entity
class Team(
    @Id @GeneratedValue
    val id: Long = 0,
    var name: String
) : BaseEntity() // BaseEntity를 상속받는다.
```

**생성되는 테이블 구조**
[그림: @MappedSuperclass를 사용한 테이블 구조]
`BaseEntity` 테이블은 생성되지 않는다. 대신, `MEMBER`와 `TEAM` 테이블에 `createdAt`, `updatedAt` 컬럼이 각각 추가된다.

**MEMBER 테이블**
| ID (PK) | USERNAME | CREATED\_AT | UPDATED\_AT |
| :--- | :--- | :--- | :--- |

**TEAM 테이블**
| ID (PK) | NAME | CREATED\_AT | UPDATED\_AT |
| :--- | :--- | :--- | :--- |

-----

`@MappedSuperclass`는 `@Inheritance`를 사용한 상속 매핑과는 완전히 다른 개념이다. 상속 매핑은 부모와 자식 테이블 간의 관계를 맺지만, `@MappedSuperclass`는 단순히 자식 클래스에게 매핑 정보만 물려줄 뿐, 테이블 간의 관계는 전혀 없다.

이 기능은 특히 Spring Data JPA의 Auditing 기능과 결합될 때 엄청난 시너지를 발휘한다. `@EntityListeners`와 함께 사용하면 `@CreatedDate`, `@LastModifiedDate` 같은 어노테이션만으로 생성/수정 시간을 자동으로 관리하는, 매우 깔끔하고 재사용성 높은 코드를 작성할 수 있다. (이 주제는 13장에서 자세히 다룬다.)

-----

이것으로 객체지향의 꽃인 상속 관계를 데이터베이스에 매핑하는 모든 전략을 마스터했다. 우리는 정규화와 성능 사이에서 줄다리기하는 조인 전략과 단일 테이블 전략을 비교했고, 사용해서는 안 될 클래스별 테이블 전략을 확인했으며, 마지막으로 코드 재사용을 위한 가장 우아한 방법인 `@MappedSuperclass`까지 배웠다.

지금까지 우리는 JPA의 세계를 지탱하는 두 개의 큰 기둥, \*\*'영속성 관리'\*\*와 \*\*'객체-테이블 매핑'\*\*을 모두 세웠다. 이제 이 견고한 토대 위에서 우리가 원하는 데이터를 자유자재로 가져오는 기술을 배울 차례다. 다음 6장부터는 SQL보다 더 객체지향적인 쿼리 언어, \*\*JPQL(Java Persistence Query Language)\*\*의 세계로 떠나보자.