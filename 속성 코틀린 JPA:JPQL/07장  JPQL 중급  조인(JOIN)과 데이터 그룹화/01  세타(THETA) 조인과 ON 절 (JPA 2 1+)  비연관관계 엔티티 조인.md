## 01\. 세타(THETA) 조인과 ON 절 (JPA 2.1+): 비연관관계 엔티티 조인

이전 절에서 배운 표준 조인은 `m.team`처럼 엔티티에 연관관계가 명시적으로 매핑된 경우에만 사용할 수 있었다. 하지만 실무에서는 전혀 연관관계가 없는 두 엔티티를 특정 조건으로 묶어서 조회해야 하는 경우가 종종 발생한다. 예를 들어, 회원의 이름(`username`)과 팀의 이름(`name`)이 같은 경우를 모두 찾아야 한다고 가정해 보자.

이처럼 연관관계가 없는 엔티티를 조인하기 위해 JPQL은 두 가지 방법을 제공한다.

-----

### **1. 세타 조인 (Theta Join)**

세타 조인은 `FROM` 절에 여러 엔티티를 나열하고, `WHERE` 절에서 이들의 조인 조건을 직접 명시하는 방식이다. SQL의 `CROSS JOIN` 후에 `WHERE`로 필터링하는 것과 유사하다.

**JPQL 문법**

```jpql
-- 회원의 이름과 팀의 이름이 같은 모든 조합을 조회
SELECT m, t FROM Member m, Team t WHERE m.username = t.name
```

`FROM` 절에 `Member`와 `Team`을 함께 명시하고, `WHERE` 절에 `m.username = t.name`이라는 조인 조건을 넣었다.

**한계**:

  * **내부 조인(INNER JOIN)만 가능**: 세타 조인으로는 외부 조인(OUTER JOIN)을 구현할 수 없다.
  * **성능 문제 가능성**: 데이터베이스에 따라 내부적으로 CROSS JOIN 후 필터링으로 동작하여 성능이 저하될 수 있다.

세타 조인은 JPQL 초기에 제공되던 기능으로, 현재는 더 강력한 대안이 존재한다.

-----

### **2. ON 절을 사용한 조인 (JPA 2.1+) - ⭐️ 권장**

JPA 2.1부터 `JOIN`문에 **`ON` 절**을 사용할 수 있게 되면서 JPQL의 조인 기능이 비약적으로 발전했다. `ON` 절을 사용하면 다음 두 가지가 가능해진다.

1.  **연관관계 없는 엔티티 외부 조인**: 세타 조인과 달리 외부 조인도 가능하다.
2.  **연관관계 있는 엔티티 조인 시 조건 추가**: 일반적인 연관관계 조인에 추가적인 필터링 조건을 더할 수 있다.

**1) 연관관계 없는 엔티티 조인**

```jpql
-- 회원의 이름과 팀의 이름이 같은 경우를 외부 조인
SELECT m, t FROM Member m LEFT JOIN Team t ON m.username = t.name
```

`FROM` 절에는 `Member`만 명시하고, `LEFT JOIN`으로 `Team`을 조인하면서 `ON` 절에 조인 조건을 직접 기술했다. 이 방식은 세타 조인보다 훨씬 직관적이고 외부 조인까지 지원하여 유연하다.

**코드 예시**

```kotlin
@Test
@Transactional
fun `ON 절을 사용한 외부 조인 테스트`() {
    val teamA = Team(name = "User1") // Member 이름과 같은 팀 이름
    entityManager.persist(teamA)
    entityManager.persist(Member(username = "User1"))
    entityManager.persist(Member(username = "User2")) // 매칭되는 팀 없음

    val jpql = "SELECT m, t FROM Member m LEFT JOIN Team t ON m.username = t.name"
    val resultList = entityManager.createQuery(jpql).resultList

    assertThat(resultList).hasSize(2) // 모든 Member가 조회됨
    // resultList[0] 에는 Member("User1")와 Team("User1")이 들어있다.
    // resultList[1] 에는 Member("User2")와 null이 들어있다.
}
```

**2) 연관관계 있는 엔티티 조인 시 조건 추가**
`ON` 절은 이미 연관관계가 있는 엔티티를 조인할 때, 그 조인 자체에 조건을 추가하는 용도로도 매우 유용하게 쓰인다. (`WHERE` 절과 미묘한 차이가 있다.)

```jpql
-- 모든 회원을 조회하되, 'TeamA'에 소속된 회원만 팀 정보를 함께 가져온다.
SELECT m, t FROM Member m LEFT JOIN m.team t ON t.name = 'TeamA'
```

만약 위 쿼리에서 `ON` 대신 `WHERE t.name = 'TeamA'`를 사용했다면, `TeamA` 소속이 아닌 회원은 아예 결과에서 제외되었을 것이다. `ON` 절은 조인 대상을 필터링하는 것이고, `WHERE` 절은 조인된 전체 결과에서 필터링하는 것이기 때문에 외부 조인에서는 그 결과가 달라진다.

-----

`ON` 절의 등장은 JPQL을 한 단계 더 성숙시켰다. 이제 개발자들은 대부분의 복잡한 조인 시나리오를 Native SQL에 의존하지 않고도 JPQL만으로 해결할 수 있게 되었다.

지금까지 배운 조인들은 SQL의 조인과 개념적으로 동일했다. 하지만 다음 절에서는 JPA에만 존재하는, 그리고 JPA 성능 최적화의 심장과도 같은 아주 특별한 조인, \*\*'페치 조인(Fetch Join)'\*\*에 대해 알아볼 것이다.