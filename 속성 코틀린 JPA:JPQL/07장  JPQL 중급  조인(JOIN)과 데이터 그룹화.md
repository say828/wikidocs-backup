# 07장: JPQL 중급: 조인(JOIN)과 데이터 그룹화

6장에서 우리는 JPQL의 기초를 탄탄히 다졌다. 단일 엔티티를 대상으로 원하는 데이터를 조회하고, DTO로 변환하며, 페이징 처리까지 자유자재로 할 수 있게 되었다. 하지만 실제 애플리케이션의 쿼리는 대부분 하나의 엔티티로 끝나지 않는다. "특정 팀에 소속된 모든 회원을 찾아라" 또는 "각 팀별 평균 회원 연령을 계산하라" 와 같이 여러 엔티티에 걸친 데이터를 조합해야 하는 경우가 허다하다.

이번 장에서는 이처럼 흩어져 있는 데이터를 하나로 묶는 강력한 기술인 \*\*조인(JOIN)\*\*에 대해 깊이 있게 탐구한다. SQL의 조인과 JPQL의 조인이 어떻게 다른지 이해하고, 가장 기본이 되는 내부 조인과 외부 조인 사용법을 익힐 것이다. 더 나아가, JPA 성능 문제의 90%를 차지하는 \*\*N+1 문제를 근본적으로 해결하는 비장의 무기, '페치 조인(Fetch Join)'\*\*의 원리와 한계까지 낱낱이 파헤친다.

마지막으로, 데이터를 특정 그룹으로 묶어 통계 정보를 추출하는 `GROUP BY`와 `HAVING` 절의 올바른 사용법을 배우며 JPQL 중급 과정을 마무리한다. 이 장을 마치면, 여러분은 단순한 단일 조회를 넘어, 복잡한 연관관계를 넘나들며 원하는 데이터를 가장 효율적으로 가져오는 실전 쿼리 전문가로 거듭나게 될 것이다.

-----

## 00\. 내부(INNER) 조인과 외부(LEFT OUTER) 조인

JPQL의 조인은 SQL의 조인과 문법은 거의 같지만, 한 가지 중요한 차이점이 있다. SQL 조인이 외래 키(FK) 값을 기준으로 테이블과 테이블을 직접 연결하는 반면, **JPQL 조인은 엔티티에 정의된 연관관계 필드(예: `m.team`)를 기반으로 객체와 객체를 연결**한다. 개발자는 더 이상 데이터베이스의 외래 키 컬럼 이름을 신경 쓸 필요 없이, 객체 그래프를 탐색하듯 자연스럽게 조인을 사용할 수 있다.

JPQL은 SQL과 마찬가지로 내부 조인과 외부 조인을 모두 지원한다.

-----

### **1. 내부 조인 (INNER JOIN)**

내부 조인은 조인하는 양쪽 엔티티에 모두 데이터가 존재하는, 즉 **연관관계가 설정된 데이터만 조회**하는 방식이다. 예를 들어, `Member`와 `Team`을 내부 조인하면, 팀에 소속된 회원만 결과에 포함된다. 팀이 없는 회원은 결과에서 제외된다.

**JPQL 문법**

  * `INNER JOIN`: 정식 문법.
  * `JOIN`: `INNER`를 생략한 축약형. 보통 축약형을 더 많이 사용한다.

<!-- end list -->

```jpql
-- 팀이 있는 회원만 조회
SELECT m FROM Member m JOIN m.team t
```

`m.team`은 `Member` 엔티티에 정의된 연관관계 필드다. `t`는 조인된 `Team` 엔티티를 가리키는 별칭이다.

**코드 예시**

```kotlin
@Test
@Transactional
fun `내부 조인 테스트`() {
    val teamA = Team(name = "Team A")
    entityManager.persist(teamA)

    entityManager.persist(Member(username = "Member1", team = teamA)) // Team A 소속
    entityManager.persist(Member(username = "Member2", team = teamA)) // Team A 소속
    entityManager.persist(Member(username = "Member3", team = null))  // 팀 없음

    val jpql = "SELECT m FROM Member m JOIN m.team t WHERE t.name = :teamName"
    val resultList = entityManager.createQuery(jpql, Member::class.java)
        .setParameter("teamName", "Team A")
        .resultList

    assertThat(resultList).hasSize(2) // 팀이 없는 Member3은 제외된다.
}
```

이 JPQL은 하이버네이트에 의해 `... FROM Member member inner join Team team on member.team_id=team.id ...` 와 같은 SQL로 번역되어 실행된다.

-----

### **2. 외부 조인 (LEFT OUTER JOIN)**

외부 조인은 조인의 왼쪽에 있는 엔티티(`Member`)를 기준으로, **연관된 데이터가 있든 없든 모두 조회**하는 방식이다. 만약 연관된 데이터(`Team`)가 없다면 그 부분은 `null`로 채워진다.

**JPQL 문법**

  * `LEFT OUTER JOIN`: 정식 문법.
  * `LEFT JOIN`: `OUTER`를 생략한 축약형.

<!-- end list -->

```jpql
-- 모든 회원을 조회하되, 팀이 있다면 팀 정보도 함께 조회
SELECT m FROM Member m LEFT JOIN m.team t
```

**코드 예시**

```kotlin
@Test
@Transactional
fun `외부 조인 테스트`() {
    val teamA = Team(name = "Team A")
    entityManager.persist(teamA)
    // ... (이전 예제와 동일한 데이터) ...

    val jpql = "SELECT m FROM Member m LEFT JOIN m.team t"
    val resultList = entityManager.createQuery(jpql, Member::class.java).resultList

    // 팀이 없는 Member3을 포함하여 모든 회원이 조회된다.
    assertThat(resultList).hasSize(3)
}
```

이 쿼리를 사용하면 소속팀이 없는 회원까지 모두 조회할 수 있다.

-----

지금까지 살펴본 조인은 엔티티에 연관관계가 명시적으로 매핑된 경우에만 사용할 수 있었다. 그렇다면 연관관계가 없는 엔티티끼리 조인하거나, 더 복잡한 조건을 조인문에 포함시키려면 어떻게 해야 할까? 다음 절에서 이 문제를 해결하는 방법을 알아보자.