## 01\. Criteria의 장점(컴파일 타임 체크)과 단점(극단적인 복잡성)

Criteria API는 JPA의 표준 기능으로서 분명한 장점과 그보다 더 명확한 단점을 동시에 가지고 있다. 어떤 도구든 그 장단점을 명확히 알아야 적재적소에 사용할 수 있다.

-----

### **장점: 컴파일 타임에 오류를 잡는 견고함**

Criteria의 유일하다시피 한, 그러나 매우 강력한 장점은 **컴파일 타임 체크**가 가능하다는 것이다. JPQL은 결국 문자열이기 때문에, 다음과 같은 실수는 애플리케이션을 실행하고 나서야 발견할 수 있다.

```jpql
// 오타 발생! username -> usernmae
"SELECT m FROM Member m WHERE m.usernmae = :name"
```

이런 런타임 오류는 애플리케이션의 안정성을 크게 해친다. 하지만 Criteria를 사용하면 모든 쿼리가 코드로 작성되므로, 필드 이름에 오타가 있거나 타입이 맞지 않는 비교를 시도하면 컴파일러가 즉시 오류를 알려준다. 이는 특히 복잡한 쿼리를 여러 개발자가 함께 유지보수해야 하는 환경에서 버그를 사전에 방지하는 든든한 안전망이 되어준다.

-----

### **단점: 유지보수를 포기하게 만드는 극단적인 복잡성**

Criteria의 장점은 명확하지만, 실무에서 개발자들이 사용을 기피하는 이유는 단점이 너무나도 치명적이기 때문이다.

**1. 극단적인 복잡성과 낮은 생산성**: 간단한 JPQL 쿼리 하나를 Criteria로 변환하려면 수많은 `CriteriaBuilder` 메서드를 조합해야 하므로 코드의 양이 최소 5\~10배 이상 늘어난다. 이는 개발 생산성을 심각하게 저하시킨다.

**2. 낮은 가독성**: 완성된 Criteria 코드는 그 자체만 보고 어떤 SQL이 생성될지 한눈에 파악하기가 매우 어렵다. 쿼리의 전체적인 구조가 여러 메서드 호출에 흩어져 있어, 코드를 위에서부터 아래로 해석하는 것이 거의 불가능에 가깝다. 이는 코드 리뷰와 유지보수를 극도로 힘들게 만든다.

**JPQL vs. Criteria 비교**
'팀 A'에 소속된 30세 이상 회원을 이름 내림차순으로 조회하는 간단한 쿼리를 비교해 보자.

**JPQL (간결하고 명확함)**

```jpql
SELECT m FROM Member m JOIN m.team t
WHERE t.name = 'Team A' AND m.age >= 30
ORDER BY m.username DESC
```

**Criteria (복잡하고 난해함)**

```kotlin
val cb = entityManager.criteriaBuilder
val cq = cb.createQuery(Member::class.java)
val m = cq.from(Member::class.java)
val t = m.join<Member, Team>("team") // 조인

val teamNamePredicate = cb.equal(t.get<String>("name"), "Team A")
val agePredicate = cb.greaterThanOrEqualTo(m.get<Int>("age"), 30)

cq.select(m)
  .where(cb.and(teamNamePredicate, agePredicate))
  .orderBy(cb.desc(m.get<String>("username")))

val resultList = entityManager.createQuery(cq).resultList
```

차이가 극명하다. 한 줄이면 될 쿼리가 열 줄이 넘는 복잡한 코드로 변했다. 이 때문에 대부분의 개발자들은 Criteria의 타입-안전성이라는 장점보다 낮은 생산성과 가독성이라는 단점이 훨씬 크다고 느낀다.

> **결론: Criteria는 실무에서 거의 사용되지 않는다.**
> 이처럼 극단적인 복잡성과 낮은 가독성 때문에, 순수한 Criteria API는 오늘날 실무에서 동적 쿼리를 작성하는 데 거의 사용되지 않는다. 대신, 개발자들은 Criteria의 타입-안전성이라는 장점은 취하면서도 훨씬 더 나은 가독성과 생산성을 제공하는 **QueryDSL**이라는 외부 라이브러리를 사용하는 것을 압도적으로 선호한다. (QueryDSL은 10장에서 자세히 다룬다.)
>
> -----

JPA는 Criteria의 단점 중 하나인 필드 이름의 문자열 사용 문제를 해결하기 위해 '메타모델'이라는 기능을 제공한다. 다음 절에서 이 기능에 대해 알아보자.