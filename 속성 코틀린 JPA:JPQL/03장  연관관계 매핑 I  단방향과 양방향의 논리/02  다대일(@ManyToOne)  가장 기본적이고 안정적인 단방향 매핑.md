## 02\. 다대일(@ManyToOne): 가장 기본적이고 안정적인 단방향 매핑

연관관계 매핑의 세계로 들어서는 가장 안전하고 확실한 첫걸음은 **다대일(N:1) 단방향** 관계를 이해하는 것이다. '다(N)'에 해당하는 `Member`가 '일(1)'에 해당하는 `Team`에 소속되는 상황을 생각해 보자. 이 관계는 데이터베이스 설계의 정석과도 같으며, JPA에서도 가장 직관적으로 매핑할 수 있다.

**단방향 관계**란 두 엔티티 중 한쪽만 상대방을 참조하는 것을 의미한다. 즉, `Member` 엔티티는 자신이 속한 `Team`을 알지만, `Team` 엔티티는 자신에게 어떤 `Member`들이 속해 있는지 전혀 알지 못한다.

-----

### **매핑 코드 살펴보기**

다대일 단방향 관계를 매핑하는 데는 `@ManyToOne`과 `@JoinColumn`이라는 두 가지 핵심 어노테이션이 사용된다.

**Member.kt (다, N)**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long = 0,
    var username: String,

    @ManyToOne(fetch = FetchType.LAZY) // 페치 전략은 반드시 LAZY로!
    @JoinColumn(name = "TEAM_ID") // MEMBER 테이블에 생성될 외래 키(FK) 컬럼 이름
    var team: Team? = null
)
```

**Team.kt (일, 1)**

```kotlin
@Entity
class Team(
    @Id @GeneratedValue
    val id: Long = 0,
    var name: String
) {
    // Member를 참조하는 필드가 전혀 없다.
}
```

  * `@ManyToOne`: 관계의 종류를 정의한다. `Member`가 '다(Many)'이고 `Team`이 '일(One)'임을 명시한다. 앞서 배운 것처럼, 페치 전략은 반드시 `FetchType.LAZY`로 설정하는 것을 습관화해야 한다.
  * `@JoinColumn(name = "TEAM_ID")`: 이 어노테이션이 실질적인 외래 키 매핑을 담당한다. '다(N)'에 해당하는 `Member` 엔티티의 테이블(`MEMBER`)에 '일(1)'에 해당하는 `Team` 엔티티의 기본 키를 저장할 **외래 키 컬럼을 생성**하라고 JPA에게 알려준다. `name` 속성으로 외래 키 컬럼의 이름을 지정할 수 있다. 만약 생략하면 JPA가 `필드명_참조하는테이블의PK컬럼명` (예: `team_id`) 규칙으로 알아서 이름을 만든다.

-----

### **저장 및 조회 로직**

이 매핑을 실제로 사용해 보자. `Member`를 생성하고 `Team`을 할당한 뒤 저장하면, JPA는 어떤 SQL을 만들어낼까?

**저장**

```kotlin
@Test
@Transactional
fun `다대일 단방향 관계 저장 테스트`() {
    val team = Team(name = "Development Team 1")
    entityManager.persist(team)

    val member = Member(username = "John")
    // 연관관계 설정: Member 객체에 Team 객체의 참조를 할당한다.
    member.team = team
    entityManager.persist(member)
}
```

위 코드가 실행되면, 트랜잭션 커밋 시점에 JPA는 `member.team` 필드를 보고 `team` 객체의 ID 값을 꺼낸다. 그리고 `MEMBER` 테이블에 `INSERT` 쿼리를 실행할 때 `TEAM_ID` 컬럼에 해당 ID 값을 포함시킨다.

**SQL 결과**

```sql
INSERT INTO TEAM (ID, NAME) VALUES (1, 'Development Team 1')
INSERT INTO MEMBER (ID, USERNAME, TEAM_ID) VALUES (10, 'John', 1)
```

**조회**

```kotlin
// 멤버 조회
val foundMember = entityManager.find(Member::class.java, 10L)

// 연관된 팀 정보 사용 (이때 Team 프록시 객체가 초기화된다)
val teamName = foundMember.team?.name 
println("Member ${foundMember.username} belongs to Team ${teamName}")
```

지연 로딩(`LAZY`)으로 설정했기 때문에, `find()` 메서드 호출 시점에는 `MEMBER` 테이블만 조회한다. 이후 `foundMember.team?.name`처럼 실제로 `Team` 객체에 접근하는 시점에 비로소 `TEAM` 테이블을 조회하는 `SELECT` 쿼리가 추가로 실행된다.

다대일 단방향 매핑은 객체지향적인 코드(`member.team = team`)와 관계형 데이터베이스의 외래 키 구조가 가장 자연스럽게 맞아떨어지는 방식이다. 그렇다면 반대로 '일(1)'인 `Team`에서 자신에게 속한 모든 `Member`를 조회하고 싶을 때는 어떻게 해야 할까? 다음 절에서 일대다 단방향 관계의 함정과 비권장 이유에 대해 알아보자.