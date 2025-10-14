## 06\. 페이징(Paging) API: `setFirstResult`와 `setMaxResults`

애플리케이션에서 게시판 목록이나 상품 목록처럼 대량의 데이터를 조회해야 할 때, 만약 검색된 데이터가 수만 건이 넘는다면 어떻게 될까? 모든 결과를 한 번에 조회하여 메모리에 올리는 것은 애플리케이션 서버에 엄청난 부하를 주며, 심각한 경우 `OutOfMemoryError`로 시스템이 멈춰버릴 수도 있다.

이런 대용량 데이터 문제를 해결하는 가장 보편적인 방법이 바로 \*\*페이징(Paging)\*\*이다. 페이징은 전체 데이터를 작은 단위의 '페이지'로 나누어, 현재 사용자가 보고 있는 페이지의 데이터만 데이터베이스에서 조회해오는 기술이다.

JPA는 어떤 데이터베이스를 사용하든 일관된 방식으로 페이징을 처리할 수 있도록 매우 간단하고 강력한 API를 제공한다.

-----

### **JPA 페이징 API**

JPA 페이징은 `Query` 객체가 제공하는 두 개의 메서드를 조합하여 간단하게 구현할 수 있다.

  * **`setFirstResult(int startPosition)`**: 조회를 시작할 위치(인덱스)를 지정한다. **인덱스는 0부터 시작한다.** (예: `10`으로 설정하면 11번째 데이터부터 조회를 시작한다.)
  * **`setMaxResults(int maxResult)`**: 한 번에 조회할 데이터의 개수(페이지의 크기)를 지정한다.

예를 들어, 한 페이지에 10개의 데이터를 보여주는 게시판의 3번째 페이지를 조회하고 싶다면, `setFirstResult(20)`과 `setMaxResults(10)`을 호출하면 된다. (21번째 데이터부터 10개를 가져온다.)

-----

### **코드로 보는 페이징 처리**

100명의 회원을 저장한 뒤, 나이를 기준으로 내림차순 정렬하여 두 번째 페이지(6번째부터 10번째까지)의 회원 5명을 조회하는 예제를 작성해 보자.

**페이징 API 실행 코드**

```kotlin
@Test
@Transactional
fun `페이징 API 테스트`() {
    // 테스트 데이터 100명 준비
    for (i in 1..100) {
        entityManager.persist(Member(username = "Member $i", age = i))
    }

    val jpql = "SELECT m FROM Member m ORDER BY m.age DESC"

    val query = entityManager.createQuery(jpql, Member::class.java)

    // 두 번째 페이지: 시작 위치는 5, 가져올 개수는 5
    val pageNumber = 2
    val pageSize = 5
    val firstResult = (pageNumber - 1) * pageSize

    query.firstResult = firstResult   // 코틀린 프로퍼티 스타일로 접근 가능
    query.maxResults = pageSize       // 코틀린 프로퍼티 스타일로 접근 가능

    val resultList: List<Member> = query.resultList

    // 결과 확인
    assertThat(resultList).hasSize(5)
    assertThat(resultList[0].age).isEqualTo(95) // 100, 99, 98, 97, 96 (첫 페이지) -> 95 (두 번째 페이지 첫 멤버)
    assertThat(resultList[4].age).isEqualTo(91) // 두 번째 페이지 마지막 멤버
}
```

**실행되는 SQL**
이 코드를 실행하면, JPA(하이버네이트)는 우리가 사용하는 데이터베이스의 방언(Dialect)에 맞춰 페이징을 위한 SQL을 자동으로 생성해준다.

  * **MySQL / PostgreSQL**: `... ORDER BY age DESC LIMIT ? OFFSET ?`
  * **Oracle**: `... ORDER BY age DESC OFFSET ? ROWS FETCH NEXT ? ROWS ONLY`

이처럼 JPA 페이징 API를 사용하면, 데이터베이스가 바뀌더라도 애플리케이션 코드를 전혀 수정할 필요 없이 페이징을 구현할 수 있다는 큰 장점이 있다.

-----

이것으로 JPQL의 기초를 모두 마스터했다. 우리는 SQL과 다른 JPQL의 객체지향적 특성을 이해했고, `SELECT`, `FROM`, `WHERE`를 사용한 기본 쿼리부터 프로젝션, DTO 변환, 안전한 파라미터 바인딩, 그리고 대용량 처리를 위한 페이징까지, 데이터를 조회하는 데 필요한 핵심 기술들을 모두 익혔다.

하지만 지금까지의 모든 쿼리는 단 하나의 엔티티(`Member`)만을 대상으로 했다. 실제 애플리케이션의 데이터는 여러 엔티티가 복잡하게 얽혀있다. 다음 7장에서는 이렇게 흩어져 있는 데이터를 하나로 모으는 강력한 기술이자, JPA 성능의 핵심인 \*\*조인(JOIN)\*\*에 대해 깊이 있게 탐구해 볼 것이다.