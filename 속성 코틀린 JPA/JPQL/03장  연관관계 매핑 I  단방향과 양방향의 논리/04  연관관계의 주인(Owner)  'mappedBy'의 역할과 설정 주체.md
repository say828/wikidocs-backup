## 04\. 연관관계의 주인(Owner): 'mappedBy'의 역할과 설정 주체

일대다 단방향 관계의 문제점을 확인했으니, 이제 가장 이상적인 해결책인 **양방향 관계**를 맺을 시간이다. 양방향 관계란 말 그대로 관계를 맺는 두 엔티티가 서로를 참조하는 것을 의미한다. 즉, `Member`는 `Team`을 알 수 있고, `Team` 역시 자신에게 속한 `Member`들의 목록을 알 수 있다.

객체지향 관점에서는 양쪽에서 서로를 참조하는 것이 아무런 문제가 되지 않는다. 하지만 테이블의 관점에서 보면 이야기가 다르다. `MEMBER` 테이블의 `TEAM_ID` 외래 키 하나가 `MEMBER -> TEAM` 관계와 `TEAM -> MEMBER` 관계를 모두 표현하고 있다.

여기서 JPA는 중대한 혼란에 빠진다. 만약 개발자가 `member.setTeam(teamA)` 라고 코드를 변경하는 동시에 `teamB.getMembers().add(member)` 라고 코드를 작성하면, JPA는 `MEMBER` 테이블의 `TEAM_ID`를 `teamA`의 ID로 바꿔야 할까, `teamB`의 ID로 바꿔야 할까? 이처럼 하나의 외래 키를 두 개의 객체 참조가 관리하려는 상황이 발생한다.

이 문제를 해결하기 위해 JPA는 \*\*연관관계의 주인(Owner of the Relationship)\*\*이라는 매우 중요한 개념을 도입했다.

-----

### **주인(Owner)과 하인(MappedBy)**

연관관계의 주인이란, 두 연관관계 중 **데이터베이스 외래 키(FK)를 직접 관리하고 등록/수정/삭제할 권한을 가진 쪽**을 의미한다. 주인만이 외래 키 값을 변경할 수 있으며, 주인이 아닌 쪽(하인)의 연관관계 설정은 데이터베이스에 전혀 반영되지 않는다. 하인은 그저 주인을 통해 설정된 관계를 조회(read-only)하는 역할만 할 뿐이다.

**주인을 정하는 규칙은 아주 명확하고 간단하다.**

> **연관관계의 주인은 외래 키가 있는 곳이다.**

`Member`와 `Team`의 관계에서 외래 키인 `TEAM_ID`는 `MEMBER` 테이블에 있다. 따라서 이 관계의 주인은 **`Member` 엔티티**다.

주인이 아닌 쪽은 자신이 하인임을 명시해야 하는데, 이때 사용하는 속성이 바로 **`mappedBy`** 다. `mappedBy`는 "나는 이 관계의 주인이 아니며, 주인의 **어떤 필드**에 의해 매핑되었다"라고 선언하는 것이다.

-----

### **가장 이상적인 양방향 매핑 코드**

이제 규칙에 따라 `Member`와 `Team`의 양방향 관계를 코드로 완성해 보자.

**Member.kt (주인, N, 외래 키가 있는 곳)**

```kotlin
@Entity
class Member(
    // ...
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "TEAM_ID") // 외래 키를 직접 매핑하므로 이쪽이 '주인'이다.
    var team: Team? = null
)
```

  * **주인**: `Member.team` 필드는 `@JoinColumn`을 통해 외래 키를 직접 관리하므로 주인이다. `mappedBy` 속성이 없다.

**Team.kt (하인, 1, 주인이 아닌 곳)**

```kotlin
@Entity
class Team(
    // ...
    // mappedBy = "team" : 이 관계는 Member 엔티티의 'team' 필드에 의해 관리된다.
    @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
    var members: MutableList<Member> = mutableListOf()
)
```

  * **하인**: `Team.members` 필드는 `@OneToMany`로 관계를 맺지만, 외래 키를 관리할 권한이 없다. `mappedBy = "team"` 속성을 통해 "이 관계의 주인은 `Member` 엔티티에 있는 `team`이라는 필드야"라고 JPA에게 알려준다. 따라서 이 필드에 `Member`를 추가하거나 삭제하는 코드는 데이터베이스에 아무런 영향을 주지 않는다.

### **연관관계의 주인에게만 값을 세팅하라**

양방향 관계에서는 항상 이 규칙을 기억해야 한다. **데이터베이스에 연관관계를 저장하고 싶다면, 반드시 주인 쪽 객체에 값을 설정해야 한다.**

```kotlin
// (WRONG) 하인에게만 값을 설정한 경우
val team = Team(name = "New Team")
val member = Member(username = "Newbie")
team.members.add(member) // DB에 반영되지 않는다!

entityManager.persist(team)
entityManager.persist(member)
// 실행 결과: MEMBER 테이블의 TEAM_ID는 null이다.
```

```kotlin
// (CORRECT) 주인에게 값을 설정한 경우
val team = Team(name = "New Team")
val member = Member(username = "Newbie")
member.team = team // 주인에게만 값을 설정해도 DB에는 정상 반영된다.

entityManager.persist(team)
entityManager.persist(member)
// 실행 결과: MEMBER 테이블의 TEAM_ID에 team의 ID가 정상적으로 들어간다.
```

`team.members.add(member)` 코드는 데이터베이스 외래 키에 아무런 영향을 주지 못한다. 오직 `member.team = team` 코드만이 `MEMBER` 테이블의 `TEAM_ID` 컬럼 값을 변경할 수 있다.

그렇다면 `team.members` 컬렉션은 왜 존재하는 걸까? 순전히 객체지향적인 관점에서, `Team` 객체를 통해 소속 `Member`들을 객체 그래프 탐색으로 조회하고 싶을 때 사용하기 위함이다. 이 객체 세상의 편의를 위해, 우리는 몇 가지 추가 작업을 해주는 것이 좋다. 다음 절에서는 양방향 관계를 더욱 안전하고 편리하게 만드는 방법에 대해 알아보자.