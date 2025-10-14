## 03\. 일대다(@OneToMany): 단방향 매핑의 함정과 비권장 이유

다대일 단방향 매핑이 `Member`에서 `Team`으로 가는 자연스러운 길이었다면, 반대로 `Team`에서 자신에게 속한 모든 `Member`의 목록을 조회하고 싶은 요구사항도 분명 존재할 것이다. 가장 먼저 떠오르는 생각은 `Team` 엔티티에 `List<Member>` 필드를 추가하는 것, 즉 **일대다(@OneToMany) 단방향** 관계를 맺는 것이다.

객체지향의 관점에서 보면 이는 지극히 당연해 보인다. `Team`이 `Member`의 컬렉션을 갖는 것은 자연스러운 모델링이다. 하지만 이 '자연스러움' 뒤에는 관계형 데이터베이스의 현실과 충돌하며 발생하는 심각한 비효율이라는 함정이 숨어있다. 결론부터 말하자면, **일대다 단방향 매핑은 실무에서 거의 사용해서는 안 되는 설계**다.

-----

### **매핑 코드와 그 이면의 문제점**

먼저 코드를 보자. `Team` 엔티티에 `@OneToMany`를 사용하여 `Member` 컬렉션을 추가한다.

**Team.kt (일, 1)**

```kotlin
@Entity
class Team(
    @Id @GeneratedValue
    val id: Long = 0,
    var name: String,

    @OneToMany(fetch = FetchType.LAZY)
    @JoinColumn(name = "TEAM_ID") // 여기가 핵심!
    var members: MutableList<Member> = mutableListOf()
)
```

**Member.kt (다, N)**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long = 0,
    var username: String,
    // Team을 참조하는 필드가 전혀 없다.
)
```

`@JoinColumn(name = "TEAM_ID")` 어노테이션의 위치가 바뀌었다. 다대일 관계에서는 '다(N)'인 `Member`에 있었지만, 여기서는 '일(1)'인 `Team`에 있다. 하지만 생각해보자. 외래 키(`TEAM_ID`)는 물리적으로 어느 테이블에 존재하는가? 당연히 '다' 쪽인 `MEMBER` 테이블에 존재한다.

JPA는 이 어노테이션을 보고 "아, `MEMBER` 테이블의 `TEAM_ID` 컬럼이 이 연관관계의 외래 키구나" 라고 인지한다. 하지만 진짜 문제는 이 관계를 **관리하고 변경하는 주체**가 `Team` 엔티티라는 점이다.

-----

### **저장 시 발생하는 비효율의 극치**

이 구조에서 새로운 `Team`과 `Member`를 함께 저장하는 코드를 실행해 보자.

```kotlin
@Test
@Transactional
fun `일대다 단방향 관계 저장 시 발생하는 문제점`() {
    val member1 = Member(username = "Messi")
    val member2 = Member(username = "Ronaldo")
    entityManager.persist(member1)
    entityManager.persist(member2)

    val team = Team(name = "World Class")
    team.members.add(member1)
    team.members.add(member2)
    entityManager.persist(team)
}
```

객체지향적으로는 완벽해 보이는 이 코드가 실행될 때, JPA는 어떤 SQL을 만들어낼까? 그 결과는 충격적이다.

**SQL 결과**

```sql
-- 1, 2. Member 저장
INSERT INTO MEMBER (ID, USERNAME) VALUES (1, 'Messi')
INSERT INTO MEMBER (ID, USERNAME) VALUES (2, 'Ronaldo')

-- 3. Team 저장
INSERT INTO TEAM (ID, NAME) VALUES (10, 'World Class')

-- 4. 연관관계 설정을 위한 추가 UPDATE 쿼리 발생!
UPDATE MEMBER SET TEAM_ID = 10 WHERE ID = 1
UPDATE MEMBER SET TEAM_ID = 10 WHERE ID = 2
```

불필요한 `UPDATE` 쿼리가 두 번이나 추가로 실행되었다. 왜 이런 일이 벌어질까?

JPA 입장에서 생각해 보자.

1.  `persist(member1)` 실행: 이 시점의 `Member` 엔티티는 자신이 어떤 `Team`에 속할지 전혀 알지 못한다. 따라서 `TEAM_ID`가 `null`인 상태로 `INSERT` 할 수밖에 없다.
2.  `persist(member2)` 실행: `member1`과 마찬가지다.
3.  `persist(team)` 실행: `Team` 엔티티를 저장한다. 이 시점에도 `Team`은 그저 `Member` 객체의 참조만 들고 있을 뿐, `MEMBER` 테이블의 `TEAM_ID` 컬럼을 직접 제어할 수는 없다.
4.  트랜잭션 커밋: JPA는 `team.members` 컬렉션을 보고 "아, `member1`과 `member2`의 `TEAM_ID`를 `team`의 ID로 설정해야 하는구나\!" 라는 것을 뒤늦게 깨닫는다. 하지만 `MEMBER` 테이블의 `INSERT`는 이미 끝났으므로, 어쩔 수 없이 `UPDATE` 쿼리를 통해 외래 키 값을 갱신하는 것이다.

-----

> **So What? (그래서 어쩌라고?)**
> 일대다 단방향 매핑은 **연관관계의 주인이 아닌 쪽에서 외래 키를 관리하려는 시도**이기 때문에 본질적인 비효율을 낳는다. 객체 모델의 편의성만 생각하고 데이터베이스의 현실을 무시한 설계다. 이 방식은 성능 저하를 유발할 뿐만 아니라, `Member`를 수정할 때도 항상 `Team` 엔티티를 거쳐야 하는 등 유지보수 측면에서도 혼란을 가중시킨다.
>
> **해결책은 간단하다. 그냥 사용하지 않으면 된다.** 대신, 이전 절에서 배운 **다대일 단방향 매핑**을 사용하라. 만약 `Team`에서도 `Member` 목록을 조회해야 한다면, 그때는 '양방향' 관계를 맺으면 된다. 다음 절에서는 바로 이 양방향 관계의 정석과 '연관관계의 주인'이라는 핵심 개념에 대해 알아볼 것이다.