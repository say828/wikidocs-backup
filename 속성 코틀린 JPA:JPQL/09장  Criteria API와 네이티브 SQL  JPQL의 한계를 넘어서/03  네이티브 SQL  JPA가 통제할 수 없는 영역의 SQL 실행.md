## 03\. 네이티브 SQL: JPA가 통제할 수 없는 영역의 SQL 실행

JPQL과 Criteria는 JPA가 제공하는 표준적이고 이식성 높은 훌륭한 쿼리 방법이다. 하지만 때로는 이들을 뛰어넘는, 데이터베이스의 밑바닥까지 내려가야만 해결할 수 있는 문제가 있다. 특정 데이터베이스에만 존재하는 강력한 함수(예: Oracle의 `CONNECT BY`), 복잡한 통계 쿼리를 위한 윈도우 함수, 성능 최적화를 위한 SQL 힌트 등은 표준 JPQL의 영역 밖이다.

이처럼 JPA가 통제할 수 없는 영역의 SQL을 실행해야 할 때, 우리는 최후의 비상 탈출구인 \*\*네이티브 SQL(Native SQL)\*\*을 사용할 수 있다. 네이티브 SQL은 JPA를 통해 순수한 SQL 문을 직접 실행하는 기능으로, 우리에게 데이터베이스의 모든 기능을 사용할 수 있는 완전한 자유를 부여한다.

-----

### **네이티브 SQL 실행하기**

`EntityManager`는 네이티브 SQL을 실행하기 위한 `createNativeQuery()` 메서드를 제공한다.

**1. 엔티티 조회**
네이티브 SQL의 실행 결과를 JPA가 관리하는 엔티티 객체로 매핑하여 받을 수 있다.

```kotlin
@Test
@Transactional
fun `네이티브 SQL로 엔티티 조회 테스트`() {
    entityManager.persist(Member(username = "Kim", age = 30))

    // 데이터베이스 테이블과 컬럼명을 직접 사용한다.
    val sql = "SELECT ID, AGE, USERNAME, TEAM_ID FROM MEMBER WHERE AGE > 20"

    // 두 번째 인자로 결과로 매핑할 엔티티 클래스를 지정한다.
    val query = entityManager.createNativeQuery(sql, Member::class.java)

    val resultList = query.resultList as List<Member>

    assertThat(resultList[0].username).isEqualTo("Kim")
}
```

**중요한 제약사항**: 이 방식이 제대로 동작하려면, `SELECT` 절에 조회하는 컬럼들이 반드시 **매핑하려는 엔티티의 모든 필드에 해당하는 컬럼**이어야 한다. 만약 `ID`나 `USERNAME` 컬럼을 빼먹고 조회하면, JPA는 `Member` 객체를 온전히 생성할 수 없어 오류를 발생시킨다. 컬럼의 순서는 상관없지만, 이름과 개수는 일치해야 한다.

**2. 값(스칼라) 조회**
엔티티가 아닌 특정 값들만 조회할 수도 있다. 이 경우, 결과는 JPQL 스칼라 프로젝션과 마찬가지로 `List<Object[]>` 또는 `List<Any?>` 형태로 반환된다.

```kotlin
val sql = "SELECT USERNAME, AGE FROM MEMBER"
val query = entityManager.createNativeQuery(sql)
val resultList = query.resultList as List<Array<Any>>
```

-----

### **장점과 단점: 자유에는 책임이 따른다**

**장점:**

  * **최고의 유연성과 성능**: 데이터베이스가 제공하는 모든 SQL 문법, 함수, 힌트를 사용하여 쿼리를 극한까지 최적화할 수 있다. JPQL로는 불가능했던 복잡한 쿼리를 해결할 수 있다.

**단점:**

  * **데이터베이스 이식성 상실**: 네이티브 SQL을 사용하는 순간, 해당 쿼리는 특정 데이터베이스(예: MySQL)에 종속된다. 만약 미래에 데이터베이스를 Oracle로 변경한다면, 모든 네이티브 SQL 코드를 수정해야 하는 재앙이 발생한다.
  * **객체지향적 추상화 파괴**: 엔티티 클래스 이름이나 필드명이 아닌, 물리적인 테이블과 컬럼명을 코드에 직접 사용하게 되므로 객체와 테이블 사이의 매핑 정보가 코드에 노출된다.
  * **유지보수의 어려움**: 엔티티의 필드명이 변경되어도 네이티브 SQL 쿼리 문자열은 자동으로 수정되지 않으므로, 개발자가 일일이 찾아 수정하지 않으면 런타임에 오류가 발생한다.

> **결론: 네이티브 SQL은 최후의 수단이다.**
> 네이티브 SQL은 강력하지만, JPA가 제공하는 이식성과 추상화라는 가장 큰 장점을 포기하는 행위다. 따라서 문제 해결을 위해 항상 **JPQL -\> JPQL 우회 전략(쿼리 분리 등) -\> 데이터베이스 뷰 -\> 네이티브 SQL** 순서로 대안을 검토해야 한다. 오직 다른 방법으로는 도저히 해결이 불가능한 복잡한 쿼리나 성능 문제에 직면했을 때만 신중하게 사용해야 한다.
>
> -----

네이티브 SQL로 엔티티가 아닌 여러 값을 조회했을 때, `Object[]`로 받는 것은 여전히 불편하다. JPA는 이 결과를 DTO로 깔끔하게 매핑할 수 있는 `@SqlResultSetMapping`이라는 고급 기능을 제공한다. 다음 절에서 자세히 알아보자.