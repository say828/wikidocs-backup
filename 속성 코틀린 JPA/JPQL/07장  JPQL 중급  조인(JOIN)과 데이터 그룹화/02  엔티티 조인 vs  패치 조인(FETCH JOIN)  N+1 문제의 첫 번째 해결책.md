## 02\. 엔티티 조인 vs. 패치 조인(FETCH JOIN): N+1 문제의 첫 번째 해결책

JPA를 사용하는 개발자라면 반드시 마주치게 되는, 그리고 반드시 해결해야 하는 성능 문제의 대명사가 바로 **N+1 문제**다. 이 문제는 지연 로딩(`LAZY`)의 순기능 이면에 숨어있는 예기치 않은 부작용이며, 페치 조인(Fetch Join)은 이 문제를 해결하는 가장 대표적이고 강력한 첫 번째 해결책이다.

-----

### **N+1 문제란 무엇인가?**

N+1 문제는 **한 번의 쿼리로 N건의 데이터를 조회했는데, 이 N건의 데이터를 사용하기 위해 N번의 추가 쿼리가 발생하는 상황**을 말한다. 이름 그대로 총 `1 + N`번의 쿼리가 실행되는 것이다.

다음 시나리오를 통해 N+1 문제가 어떻게 발생하는지 살펴보자.

1.  **팀 A와 팀 B가 있고, 각각 2명의 회원이 소속되어 있다고 가정하자.** (`총 4명의 회원`)

2.  **모든 팀을 조회한다. (쿼리 1번)**

    ```kotlin
    val jpql = "SELECT t FROM Team t"
    val teams = entityManager.createQuery(jpql, Team::class.java).resultList // 1번의 SELECT 쿼리 실행
    ```

    이 쿼리로 `Team` 엔티티 2개(팀 A, 팀 B)가 영속성 컨텍스트에 로드된다. `Team`의 `members` 컬렉션은 지연 로딩(`LAZY`)이므로, 이 시점까지는 `Member` 데이터가 로딩되지 않는다.

3.  **각 팀의 회원 이름을 출력한다. (쿼리 N번)**

    ```kotlin
    for (team in teams) {
        println("Team: ${team.name}")
        // LAZY 로딩된 members 컬렉션을 최초로 사용하는 순간, 추가 쿼리가 발생한다!
        for (member in team.members) {
            println(" - Member: ${member.username}")
        }
    }
    ```

      * 첫 번째 루프에서 `teams[0]`(팀 A)의 `team.members`에 접근하는 순간, 팀 A에 소속된 회원들을 조회하기 위한 `SELECT * FROM Member WHERE team_id = ?` 쿼리가 실행된다.
      * 두 번째 루프에서 `teams[1]`(팀 B)의 `team.members`에 접근하는 순간, 팀 B에 소속된 회원들을 조회하기 위한 `SELECT * FROM Member WHERE team_id = ?` 쿼리가 또 실행된다.

**결과**: 팀을 조회하기 위한 쿼리 1번 + 각 팀별 회원을 조회하기 위한 쿼리 N번(여기서는 2번) = 총 **1+N** 번의 쿼리가 데이터베이스로 전송된다. 만약 팀이 100개였다면, 101번의 쿼리가 실행되는 끔찍한 성능 재앙이 발생한다.

-----

### **해결책: 페치 조인 (FETCH JOIN)**

이 N+1 문제를 해결하기 위해 JPA는 **페치 조인**이라는 특별한 기능을 제공한다. 페치 조인은 JPQL에서만 사용할 수 있는 기능으로, SQL의 조인 종류는 아니다. 그 목적은 단 하나, **JPQL로 특정 엔티티를 조회할 때, 그와 연관된 엔티티나 컬렉션을 함께 DB에서 조회하여 프록시가 아닌 실제 객체로 채워주는 것**이다.

**일반 조인 vs. 페치 조인**

  * **일반 조인**: `SELECT t FROM Team t JOIN t.members m`
      * 쿼리 결과로 `Team` 엔티티만 반환된다. `t.members`는 여전히 초기화되지 않은 프록시 컬렉션이다. 따라서 N+1 문제가 **해결되지 않는다.** 일반 조인은 단순히 `WHERE`나 `SELECT` 절에서 조인된 대상을 사용하기 위한 목적일 뿐이다.
  * **페치 조인**: `SELECT t FROM Team t JOIN FETCH t.members m`
      * `JOIN` 뒤에 `FETCH` 키워드 하나만 추가하면 된다.
      * 이 쿼리는 `Team`을 조회하면서, 관련된 `Member` 데이터까지 **모두 한 번의 `JOIN` 쿼리로 가져온다.**
      * 쿼리 결과로 반환된 `Team` 엔티티의 `t.members` 필드는 더 이상 프록시가 아닌, 실제 `Member` 객체들로 채워진 컬렉션이 된다.

**페치 조인 적용 코드**

```kotlin
@Test
@Transactional
fun `페치 조인으로 N+1 문제 해결`() {
    // ... (데이터 설정은 위와 동일) ...

    println("--- 페치 조인 쿼리 실행 ---")
    val jpql = "SELECT t FROM Team t JOIN FETCH t.members"
    val teams = entityManager.createQuery(jpql, Team::class.java).resultList

    println("--- 조회된 데이터 사용 ---")
    for (team in teams) {
        println("Team: ${team.name}, Member Count: ${team.members.size}")
        for (member in team.members) {
            println(" - Member: ${member.username}")
        }
    }
}
```

**SQL 결과**

```sql
--- 페치 조인 쿼리 실행 ---
SELECT ... 
FROM 
    Team t1_0 
INNER JOIN 
    Member m1_0 ON t1_0.id=m1_0.team_id
--- 조회된 데이터 사용 ---
Team: Team A, Member Count: 2
 - Member: Member1
 - Member: Member2
Team: Team B, Member Count: 2
 - Member: Member3
 - Member: Member4
```

결과를 보면 명확하다. 단 **한 번의 `JOIN` SQL**로 모든 `Team`과 `Member` 데이터를 가져왔고, 이후 `team.members`를 사용할 때 아무런 추가 쿼리도 발생하지 않았다. N+1 문제가 완벽하게 해결된 것이다.

-----

페치 조인은 JPA 성능 튜닝의 시작이자 끝이라고 할 수 있을 만큼 중요하고 강력한 기능이다. 하지만 이 강력한 기능에도 몇 가지 제약과 한계가 존재한다. 특히 컬렉션을 대상으로 페치 조인을 사용할 때는 예상치 못한 문제에 부딪힐 수 있다. 다음 절에서 그 한계에 대해 자세히 알아보자.