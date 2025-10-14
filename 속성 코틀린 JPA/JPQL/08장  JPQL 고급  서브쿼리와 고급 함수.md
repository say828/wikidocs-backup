# 08장: JPQL 고급: 서브쿼리와 고급 함수

지금까지 우리는 JPQL의 기초부터 중급까지, 데이터를 조회하는 데 필요한 대부분의 기술을 연마했다. 단일 엔티티 조회부터 시작하여 여러 엔티티를 묶는 조인, N+1 문제를 해결하는 페치 조인, 그리고 데이터를 요약하는 그룹화까지, 이제 여러분은 꽤 정교한 쿼리를 작성할 수 있는 능력을 갖추게 되었다.

이번 장에서는 JPQL의 마지막 여정으로, 지금까지 배운 기술들을 한 차원 더 높은 수준으로 끌어올리는 고급 기능들을 탐험한다. 우리는 쿼리 안에 또 다른 쿼리를 넣어 복잡한 조건을 해결하는 **서브쿼리**의 활용법과 그 명확한 한계를 배울 것이다. 또한, 상속 관계에 있는 엔티티를 유연하게 다루는 **다형성 쿼리**, JPQL이 기본으로 제공하는 유용한 **내장 함수들**, 그리고 데이터베이스의 특정 함수를 직접 호출하는 **사용자 정의 함수**까지 알아본다.

마지막으로, 수백, 수천 건의 데이터를 한 번에 수정하거나 삭제해야 할 때 압도적인 성능을 발휘하는 \*\*벌크 연산(Bulk Operation)\*\*의 원리와 주의점까지 마스터한다. 이 장을 마치면, 여러분은 JPQL이라는 언어의 잠재력을 거의 100% 활용하여, 어떤 복잡한 데이터 요구사항에도 자신감 있게 대처하는 진정한 JPQL 전문가가 될 것이다.

-----

## 00\. 서브쿼리(Subquery)의 한계와 활용: WHERE, HAVING 절에서의 사용

\*\*서브쿼리(Subquery)\*\*는 하나의 JPQL 문 안에 포함된 또 다른 JPQL 문을 의미한다. 메인 쿼리가 실행되기 전에 내부적으로 먼저 실행되며, 그 결과는 메인 쿼리의 조건 등으로 활용된다. 서브쿼리를 사용하면 쿼리를 여러 번 실행해서 애플리케이션 레벨에서 조합해야 했던 복잡한 로직을 단 한 번의 쿼리로 해결할 수 있다.

-----

### **JPQL 서브쿼리의 명확한 한계**

서브쿼리를 배우기에 앞서, JPQL 서브쿼리의 매우 중요한 한계점을 먼저 인지해야 한다. SQL에서는 `SELECT`, `FROM`, `WHERE`, `HAVING` 등 거의 모든 곳에서 서브쿼리를 사용할 수 있지만, **JPA 표준 명세상 JPQL 서브쿼리는 `WHERE` 절과 `HAVING` 절에서만 사용이 가능**하다.

`FROM` 절의 서브쿼리(인라인 뷰)나 `SELECT` 절의 서브쿼리(스칼라 서브쿼리)는 표준 JPQL에서 지원하지 않는다. (물론 하이버네이트 6+ 부터는 일부 제한적으로 지원하지만, 이식성을 고려한다면 주의해야 한다.)

-----

### **`WHERE` 절에서의 서브쿼리 활용**

`WHERE` 절에서는 `[NOT] IN`, `[NOT] EXISTS`, `ANY`, `ALL`, `SOME` 등의 연산자와 함께 서브쿼리를 사용하여 복잡한 필터링 조건을 만들 수 있다.

**1. `[NOT] IN`**: 서브쿼리의 결과 집합에 포함되거나 포함되지 않는 데이터를 조회한다.

```jpql
-- 예제: 한 번이라도 주문(Order)한 이력이 있는 회원(Member)을 모두 조회
SELECT m FROM Member m
WHERE m.id IN (SELECT o.member.id FROM Order o)
```

**2. `[NOT] EXISTS`**: 서브쿼리의 결과가 하나라도 존재하는지(존재하지 않는지) 여부를 확인한다.

```jpql
-- 예제: 팀에 소속된 회원이 한 명이라도 있는 팀(Team)을 조회
SELECT t FROM Team t
WHERE EXISTS (SELECT m FROM Member m WHERE m.team = t)
```

`EXISTS`는 보통 `IN`보다 성능상 이점이 있는 경우가 많다.

**3. `ALL`, `ANY`(또는 `SOME`)**: 서브쿼리가 반환하는 모든(ALL) 결과 또는 일부(ANY, SOME) 결과와 비교한다.

```jpql
-- 예제: 전체 상품(Product) 각각의 재고가, 모든 상품의 평균 재고보다 많은 상품만 조회
SELECT p FROM Product p
WHERE p.stockAmount > ALL (SELECT p2.stockAmount FROM Product p2 WHERE p2.category = 'Electronics')
```

위 쿼리는 `p.stockAmount`가 'Electronics' 카테고리에 있는 모든 상품의 `stockAmount`보다 클 때 참이 된다.

-----

### **`HAVING` 절에서의 서브쿼리 활용**

`HAVING` 절에서도 서브쿼리를 사용하여 그룹화된 결과에 대한 복잡한 조건을 추가할 수 있다.

```jpql
-- 예제: 평균 주문 금액이 1000원 이상인 카테고리 조회
SELECT o.product.category, AVG(o.orderPrice)
FROM OrderItem o
GROUP BY o.product.category
HAVING AVG(o.orderPrice) > (SELECT MIN(p.price) FROM Product p WHERE p.stockAmount < 10)
```

위 쿼리는 그룹화된 평균 주문 금액이, 재고 10개 미만인 상품들의 최저가보다 높은 카테고리만 필터링한다.

이처럼 서브쿼리는 `WHERE`와 `HAVING` 절에서 매우 강력한 필터링 도구로 사용된다. 하지만 `FROM` 절에서 사용할 수 없다는 한계는 어떻게 극복해야 할까? 다음 절에서 이 문제에 대한 우회 전략을 알아보자.