## 05\. 양방향 관계의 정석: `toString()`, `hashCode()` 순환 참조 문제 해결과 편의 메서드

연관관계의 주인을 정하고 주인에게만 값을 설정하면 데이터베이스에는 관계가 정상적으로 반영된다. 하지만 아직 한 가지 문제가 남아있다. 바로 **객체 상태의 불일치**다.

다음 코드를 보자.

```kotlin
val team = Team(name = "Real Madrid")
val member = Member(username = "Benzema")

// 주인인 Member에만 연관관계를 설정했다.
member.team = team

entityManager.persist(team)
entityManager.persist(member)

// DB에는 정상이지만, 객체 세상에서는?
println(team.members.contains(member)) // false!
```

분명 `member`는 `team`에 소속시켰고, 데이터베이스의 `MEMBER` 테이블에도 `TEAM_ID`가 정상적으로 들어갔다. 하지만 `team.members` 컬렉션에는 `member`가 존재하지 않는다. 1차 캐시에 살아있는 객체들의 상태가 불일치하는 것이다. 물론 영속성 컨텍스트를 `clear()`하고 다시 조회하면 `team.members`에 `member`가 포함되어 있겠지만, 현재 트랜잭션 안에서는 데이터가 맞지 않다.

이런 실수를 방지하고, 코드를 더 객체지향적으로 만들기 위해 우리는 \*\*연관관계 편의 메서드(Convenience Method)\*\*를 사용하는 것이 좋다.

-----

### **연관관계 편의 메서드**

편의 메서드는 양쪽의 참조를 한 번에 설정해 주는 메서드다. 이 메서드는 연관관계의 주인 쪽에 만드는 것이 일반적이지만, 양쪽 어디에 만들어도 상관은 없다. 중요한 것은 양쪽 관계를 항상 원자적으로(atomically) 묶어주는 것이다.

**Member.kt (주인)**

```kotlin
// ...
var team: Team? = null
    private set // 외부에서 team을 직접 변경하는 것을 막는다.

// 연관관계 편의 메서드
fun changeTeam(team: Team) {
    // 1. 기존 팀이 있다면, 기존 팀의 members 컬렉션에서 현재 member를 제거한다.
    this.team?.members?.remove(this)

    // 2. 새로운 팀을 현재 member의 team으로 설정한다. (주인에게 값 설정)
    this.team = team

    // 3. 새로운 팀의 members 컬렉션에 현재 member를 추가한다. (하인에게 값 설정)
    team.members.add(this)
}
// ...
```

이제부터 `member`의 팀을 변경할 때는 `member.team = newTeam` 대신 `member.changeTeam(newTeam)`을 호출하면 된다. 이렇게 하면 `member.team` 참조와 `team.members` 컬렉션의 상태가 항상 일관성을 유지하게 되어 실수를 원천적으로 방지할 수 있다.

-----

### **양방향 관계의 무한 루프 함정**

양방향 관계를 맺을 때 개발자들이 가장 많이 겪는 문제는 \*\*무한 루프(Infinite Loop)\*\*다. 특히 `toString()`, `hashCode()`, 그리고 JSON 직렬화 과정에서 `StackOverflowError`를 마주하게 될 확률이 매우 높다.

**1. `toString()` 무한 루프**
`Member`의 `toString()`을 호출하면, `member.team`의 `toString()`을 호출한다. `Team`의 `toString()`은 `team.members` 리스트의 각 `Member`에 대해 `toString()`을 다시 호출한다. 이 과정이 무한 반복되며 스택 오버플로우가 발생한다.

  * **해결책**: 양쪽 엔티티 중 한쪽의 `toString()`에서 연관관계 필드를 출력하지 않도록 수정해야 한다. 보통 컬렉션을 가진 `@OneToMany` 쪽에서 제외하는 것이 일반적이다.

**2. JSON 직렬화 무한 루프**
`toString()`과 똑같은 문제가 API 응답을 위해 엔티티를 JSON으로 변환할 때도 발생한다. Jackson 라이브러리가 `Member`를 JSON으로 바꾸려다 `team`을 만나고, 다시 `team`을 바꾸려다 `members` 리스트를 만나면서 무한 루프에 빠진다.

  * **해결책**: `@JsonIgnore` 어노테이션을 사용하여 한쪽의 연관관계 필드가 직렬화에 포함되지 않도록 해야 한다. DTO(Data Transfer Object)를 사용하여 엔티티를 직접 노출하지 않는 것이 더 근본적인 해결책이다.

-----

이것으로 연관관계 매핑의 기본기를 모두 다졌다. 우리는 객체와 테이블의 근본적인 차이를 이해했고, EAGER와 LAZY라는 페치 전략을 배웠다. 가장 안정적인 다대일 단방향 관계를 마스터했고, 일대다 단방향 관계의 함정을 피하는 법을 알게 되었으며, 마침내 '연관관계의 주인'과 `mappedBy`를 통해 가장 이상적인 양방향 관계를 설정하는 방법까지 터득했다.

이제 이 지식을 바탕으로 조금 더 복잡한 관계의 세계로 나아갈 준비가 되었다. 다음 4장에서는 일대일(@OneToOne) 관계의 미묘한 차이점, 실무에서 절대 사용하면 안 되는 다대다(@ManyToMany) 관계와 그 대안, 그리고 JPA의 강력한 기능인 영속성 전이(Cascade)와 고아 객체(Orphan Removal)에 대해 깊이 있게 탐구해 볼 것이다.