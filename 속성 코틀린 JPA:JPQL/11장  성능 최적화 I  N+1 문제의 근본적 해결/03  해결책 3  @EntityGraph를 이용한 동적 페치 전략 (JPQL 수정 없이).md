## 03\. 해결책 3: @EntityGraph를 이용한 동적 페치 전략 (JPQL 수정 없이)

페치 조인은 JPQL 쿼리에 직접 `JOIN FETCH`를 명시하여 N+1 문제를 해결하는, 매우 명시적이고 강력한 방법이다. 하지만 이 방식은 때로는 유연성이 부족하게 느껴질 수 있다. 같은 JPQL 쿼리라도 어떤 비즈니스 로직에서는 연관된 엔티티를 함께 가져오고 싶고(EAGER처럼), 다른 로직에서는 따로 가져오고 싶을 때(LAZY처럼)가 있다. 이때마다 `JOIN FETCH`가 있고 없는 두 가지 버전의 JPQL을 모두 만들어야 할까?

이러한 딜레마를 해결하기 위해 JPA 2.1부터 \*\*`@EntityGraph`\*\*라는 세련된 기능이 추가되었다. `@EntityGraph`는 **JPQL은 전혀 수정하지 않으면서, 특정 쿼리를 실행할 때 함께 조회할 연관 엔티티를 동적으로 지정**할 수 있게 해주는 기능이다. 즉, 쿼리 실행 시점에 페치 전략을 동적으로 선택하는 것이다.

-----

### **`@EntityGraph` 사용법**

`@EntityGraph`는 주로 스프링 데이터 JPA 리포지토리와 함께 사용될 때 그 진가가 드러난다.

**1단계: 엔티티에 `@NamedEntityGraph` 정의**
먼저, 엔티티 클래스 위에 `@NamedEntityGraph` 어노테이션을 사용하여 함께 조회할 속성(attribute)의 묶음에 이름을 붙여 정의한다.

**Member.kt**

```kotlin
@Entity
// "Member.withTeam" 이라는 이름의 엔티티 그래프를 정의한다.
// 이 그래프는 'team' 속성을 함께 가져오도록 설정한다.
@NamedEntityGraph(name = "Member.withTeam", attributeNodes = [
    NamedAttributeNode("team")
])
class Member(
    @Id @GeneratedValue
    val id: Long = 0,
    var username: String,

    @ManyToOne(fetch = FetchType.LAZY) // 기본 페치 전략은 LAZY
    @JoinColumn(name = "TEAM_ID")
    var team: Team? = null
)
```

  * `@NamedEntityGraph`: 엔티티 그래프에 이름을 부여한다.
  * `attributeNodes`: 함께 조회할 필드의 이름을 `NamedAttributeNode`로 지정한다. 여기서는 `team` 필드를 지정했다.

**2단계: 리포지토리 메서드에 `@EntityGraph` 적용**
이제 스프링 데이터 JPA 리포지토리의 메서드에 `@EntityGraph` 어노테이션을 붙여, 위에서 정의한 엔티티 그래프를 사용하겠다고 알려주기만 하면 된다.

**MemberRepository.kt**

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {

    // JPQL은 일반적인 형태 그대로 둔다. JOIN FETCH가 없다!
    @Query("SELECT m FROM Member m")
    // 이 쿼리를 실행할 때 "Member.withTeam" 엔티티 그래프를 사용하라고 지시한다.
    @EntityGraph(value = "Member.withTeam")
    fun findAllWithTeam(): List<Member>

    // 쿼리 메서드에도 적용 가능하다.
    @EntityGraph(attributePaths = ["team"]) // 직접 속성 경로를 지정할 수도 있다.
    override fun findAll(): List<Member>
}
```

  * `@EntityGraph(value = "...")`: 사용할 `@NamedEntityGraph`의 이름을 지정한다.
  * `@EntityGraph(attributePaths = ["..."])`: `@NamedEntityGraph`를 미리 정의하지 않고, 메서드에 직접 함께 조회할 필드 경로를 명시할 수도 있다.

**동작 결과**
`findAllWithTeam()` 메서드를 호출하면, 비록 JPQL에는 `JOIN FETCH`가 없지만, JPA는 `@EntityGraph` 설정을 보고 `Member`를 조회할 때 `Team`을 \*\*`OUTER JOIN`\*\*하는 SQL을 자동으로 생성하여 실행한다. N+1 문제가 깔끔하게 해결되는 것이다. 만약 `@EntityGraph`가 없는 다른 메서드(`findByUsername(...)` 등)를 호출하면, `Member.team`은 다시 기본 전략인 `LAZY`로 동작한다.

-----

### **`EntityGraph.EntityGraphType` (FETCH vs. LOAD)**

`@EntityGraph` 어노테이션에는 `type`이라는 속성이 있다.

  * **`EntityGraph.EntityGraphType.FETCH` (기본값)**: 엔티티 그래프에 명시된 속성들은 EAGER로, 나머지 속성들은 기본 페치 전략(LAZY 등)으로 조회한다.
  * **`EntityGraph.EntityGraphType.LOAD`**: 엔티티 그래프에 명시된 속성들만 EAGER로 조회하고, **나머지 모든 속성들은 LAZY로 강제**된다. `@ManyToOne`의 기본 전략이 `EAGER`인 경우에도 `LOAD`를 사용하면 `LAZY`로 동작하게 만들 수 있다.

`@EntityGraph`는 JPQL 쿼리의 순수성을 유지하면서 성능 최적화를 동적으로 적용할 수 있는 매우 우아한 방법이다. 페치 조인이 쿼리와 최적화 로직이 강하게 결합된 방식이라면, `@EntityGraph`는 그 둘의 책임을 분리(Separation of Concerns)하는 세련된 접근법이라 할 수 있다.

이제 N+1 문제를 해결하는 세 가지 핵심 무기를 모두 배웠다. 하지만 때로는 우리가 의도치 않은 곳에서 N+1 문제가 발생하는 경우가 있다. 다음 절에서는 많은 개발자들을 괴롭히는 `OSIV`의 진실에 대해 파헤쳐 본다.