## 04\. GROUP BY와 HAVING: 집계 함수(`COUNT`, `SUM`, `AVG` 등)의 올바른 사용

조인은 데이터를 수평적으로 확장하는 기술이었다면, \*\*`GROUP BY`\*\*와 \*\*`HAVING`\*\*은 데이터를 수직적으로 요약하고 분석하는 기술이다. JPQL은 SQL과 거의 동일한 방식으로 그룹화와 집계 함수를 지원하여, 통계 데이터를 효과적으로 조회할 수 있게 해준다.

-----

### **집계 함수 (Aggregate Functions)**

JPQL에서 사용할 수 있는 주요 집계 함수는 다음과 같다.

  * `COUNT(x)`: 결과의 개수를 센다. `COUNT(*)`는 지원하지 않으며, `COUNT(m)`처럼 별칭을 사용해야 한다.
  * `SUM(x)`: 숫자 타입 필드의 합계를 계산한다.
  * `AVG(x)`: 숫자 타입 필드의 평균을 계산한다.
  * `MAX(x)`: 필드의 최댓값을 구한다.
  * `MIN(x)`: 필드의 최솟값을 구한다.

-----

### **GROUP BY**

`GROUP BY` 절은 특정 필드를 기준으로 데이터를 그룹화하고, 각 그룹에 대해 집계 함수를 적용하는 데 사용된다. 예를 들어, 각 팀별로 소속된 회원의 수를 계산할 수 있다.

**JPQL 문법**

```jpql
-- 각 팀의 이름과 해당 팀의 회원 수를 조회
SELECT t.name, COUNT(m.id)
FROM Member m JOIN m.team t
GROUP BY t.name
```

`GROUP BY` 절에 명시된 `t.name`을 기준으로 데이터를 묶고, 각 그룹에 대해 `COUNT(m.id)`를 계산한다.

-----

### **HAVING**

`HAVING` 절은 **`GROUP BY`로 그룹화된 결과에 대해 추가적인 필터링 조건**을 적용할 때 사용한다. `WHERE` 절과 비슷해 보이지만, 동작 시점이 완전히 다르다.

  * **`WHERE`**: 그룹화하기 **전**의 원본 데이터를 필터링한다.
  * **`HAVING`**: 그룹화가 완료된 **후**의 집계 결과를 필터링한다. 따라서 `HAVING` 절에는 `COUNT`, `SUM` 같은 집계 함수를 조건으로 사용할 수 있다.

**JPQL 문법**

```jpql
-- 회원 수가 10명 이상인 팀만 조회
SELECT t.name, COUNT(m.id)
FROM Member m JOIN m.team t
GROUP BY t.name
HAVING COUNT(m.id) >= 10
```

`GROUP BY`로 각 팀의 회원 수를 계산한 뒤, 그 결과(`COUNT(m.id)`)가 10 이상인 그룹만 최종 결과에 포함시킨다.

-----

### **코드 예시**

팀별 평균 연령이 25세 이상인 팀의 이름과 평균 연령을 조회하는 예제를 작성해 보자.

```kotlin
@Test
@Transactional
fun `GROUP BY와 HAVING 테스트`() {
    val teamA = Team(name = "Team A")
    val teamB = Team(name = "Team B")
    entityManager.persist(teamA)
    entityManager.persist(teamB)

    entityManager.persist(Member(username = "User1", age = 20, team = teamA))
    entityManager.persist(Member(username = "User2", age = 30, team = teamA)) // Team A 평균: 25

    entityManager.persist(Member(username = "User3", age = 30, team = teamB))
    entityManager.persist(Member(username = "User4", age = 40, team = teamB)) // Team B 평균: 35

    val jpql = """
        SELECT t.name, AVG(m.age)
        FROM Member m JOIN m.team t
        GROUP BY t.name
        HAVING AVG(m.age) >= 25.0
    """

    val resultList = entityManager.createQuery(jpql).resultList as List<Array<Any>>

    assertThat(resultList).hasSize(2)
    // 결과 순서는 보장되지 않으므로, 내용만 확인
    val resultMap = resultList.associate { (it[0] as String) to (it[1] as Double) }
    assertThat(resultMap["Team A"]).isEqualTo(25.0)
    assertThat(resultMap["Team B"]).isEqualTo(35.0)
}
```

-----

이것으로 JPQL의 중급 과정을 마친다. 우리는 연관관계가 있는 데이터를 자유자재로 묶는 다양한 조인 기법을 배웠고, N+1 문제의 핵심 해결책인 페치 조인의 원리와 한계를 파악했으며, 데이터를 요약하고 분석하는 집계 기능까지 마스터했다.

지금까지 배운 내용만으로도 대부분의 실무적인 조회 쿼리는 충분히 작성할 수 있다. 하지만 JPQL은 여기서 멈추지 않는다. 다음 8장에서는 서브쿼리, 다형성 쿼리, 벌크 연산 등 JPQL의 더욱 강력하고 깊이 있는 고급 기능들을 탐험하며 JPQL의 완전한 정복을 향해 나아갈 것이다.