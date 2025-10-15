## 01\. FROM 절 서브쿼리의 제약과 인라인 뷰(Inline View) 우회 전략

이전 절에서 JPQL 서브쿼리는 `WHERE`, `HAVING` 절에서만 사용할 수 있다는 한계를 언급했다. 이 제약이 가장 아쉽게 느껴지는 순간은 바로 **`FROM` 절의 서브쿼리, 즉 인라인 뷰(Inline View)를 사용할 수 없을 때**다.

SQL에서는 `FROM` 절에 서브쿼리를 넣어, 그 결과를 하나의 가상 테이블처럼 사용하고 다시 조인하는 복잡한 로직을 자주 사용한다. 예를 들어, '각 팀별로 가장 나이가 많은 회원을 한 명씩 뽑아서, 그 회원들의 정보를 조회'하는 쿼리는 인라인 뷰 없이는 작성하기 매우 까다롭다.

```sql
-- SQL에서는 가능하지만, JPQL에서는 불가능한 쿼리
SELECT m.*, t.name
FROM (
    SELECT member_id, ROW_NUMBER() OVER (PARTITION BY team_id ORDER BY age DESC) as rn
    FROM member
) as ranked_member
JOIN member m ON ranked_member.member_id = m.id
JOIN team t ON m.team_id = t.id
WHERE ranked_member.rn = 1;
```

이처럼 강력한 인라인 뷰를 JPQL에서는 표준적으로 지원하지 않는다. 그렇다면 우리는 이 문제를 어떻게 해결해야 할까? 포기하고 모든 로직을 애플리케이션 메모리에서 처리해야 할까? 다행히 몇 가지 효과적인 우회 전략이 존재한다.

-----

### **우회 전략 1: 쿼리를 두 번으로 분리**

가장 간단하고 JPQL의 순수성을 유지하는 방법이다. 복잡한 쿼리 하나를 두 개의 간단한 쿼리로 나누어 실행하는 것이다.

예를 들어, '평균 나이보다 나이가 많은 회원'을 조회한다고 가정해 보자.

1.  **첫 번째 쿼리**: 먼저 모든 회원의 평균 나이를 계산한다.
    ```kotlin
    val avgAge = entityManager.createQuery("SELECT AVG(m.age) FROM Member m", Double::class.java).singleResult
    ```
2.  **두 번째 쿼리**: 계산된 평균 나이 값을 파라미터로 사용하여, 조건에 맞는 회원들을 조회한다.
    ```kotlin
    val jpql = "SELECT m FROM Member m WHERE m.age > :avgAge"
    val resultList = entityManager.createQuery(jpql, Member::class.java)
        .setParameter("avgAge", avgAge)
        .resultList
    ```

이 방식은 DB와의 통신이 한 번 더 발생한다는 단점이 있지만, JPQL만으로 해결할 수 있어 데이터베이스 독립성을 유지하고 코드를 이해하기 쉽게 만든다.

-----

### **우회 전략 2: 네이티브 SQL (Native SQL) 사용**

도저히 JPQL로 해결할 수 없거나, 특정 데이터베이스의 고유 기능(계층형 쿼리, 윈도우 함수 등)을 사용해야만 하는 복잡한 쿼리는 과감하게 **네이티브 SQL**을 사용하는 것이 현명한 선택이다.

JPA는 `entityManager.createNativeQuery()` 메서드를 통해 순수한 SQL을 직접 실행하고, 그 결과를 엔티티나 DTO로 매핑하는 기능을 제공한다.

```kotlin
val sql = "SELECT * FROM MEMBER WHERE ..." // 복잡한 인라인 뷰가 포함된 SQL
val query = entityManager.createNativeQuery(sql, Member::class.java)
val resultList = query.resultList
```

이 방법은 JPQL의 한계를 뛰어넘어 데이터베이스의 모든 기능을 활용할 수 있게 해주지만, 특정 데이터베이스에 종속적인 쿼리가 되어 이식성이 떨어진다는 단점이 있다.

-----

### **우회 전략 3: 데이터베이스 뷰(View) 활용**

자주 사용되는 복잡한 쿼리라면, 아예 데이터베이스에 \*\*뷰(View)\*\*를 생성하고, 해당 뷰를 하나의 테이블처럼 매핑한 엔티티를 만드는 것도 훌륭한 전략이다.

1.  DB에 복잡한 인라인 뷰 쿼리를 `MEMBER_VIEW`라는 이름의 뷰로 생성한다.
2.  `@Entity @Table(name = "MEMBER_VIEW")` 와 같이 뷰를 테이블처럼 매핑하는 읽기 전용 엔티티를 만든다.
3.  이제 JPQL에서는 이 뷰 엔티티를 일반 엔티티처럼 간단하게 조회할 수 있다. `SELECT v FROM MemberView v`

> **So What? (그래서 어쩌라고?)**
> JPQL의 `FROM` 절 서브쿼리 제약은 분명한 한계지만, 해결책은 존재한다.
>
>   * **우선 쿼리를 두 번으로 분리하는 방법을 시도하라.** 이것이 가장 JPA다운 해결책이다.
>   * **분리가 불가능하거나 성능이 매우 중요한 통계/분석 쿼리라면, 주저 없이 네이티브 SQL을 사용하라.** ORM의 편리함보다 데이터의 정확성과 성능이 우선이다.
>   * **자주 사용되는 고정된 형태의 복잡한 조회라면, 뷰를 만드는 것이 가장 깔끔한 아키텍처다.**

이제 JPQL의 한계를 극복하는 법을 배웠다. 다음 절에서는 반대로 JPQL만이 가진 객체지향적 특징인 '다형성 쿼리'에 대해 알아보자.