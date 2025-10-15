## 05\. 벌크 연산(UPDATE, DELETE): 영속성 컨텍스트 우회와 2차 캐시 동기화 문제

지금까지 우리가 배운 JPQL은 모두 조회(SELECT)에 관한 것이었다. 그렇다면 대량의 데이터를 한 번에 수정하거나 삭제해야 할 때는 어떻게 해야 할까? 예를 들어, 모든 상품의 가격을 10% 인상하거나, 특정 기간이 지난 비활성 회원을 모두 삭제하는 경우다.

가장 순진한 방법은 수정/삭제할 모든 엔티티를 조회한 뒤, 루프를 돌며 하나씩 수정(변경 감지 활용)하거나 `entityManager.remove()`를 호출하는 것이다. 하지만 데이터가 수만 건이라면, 수만 개의 엔티티를 메모리에 로드하고 수만 번의 `UPDATE` 또는 `DELETE` 쿼리를 실행하는 것은 끔찍한 성능 저하를 유발한다.

이 문제를 해결하기 위해 JPQL은 데이터베이스에 직접 `UPDATE`와 `DELETE`를 실행할 수 있는 **벌크 연산(Bulk Operation)** 기능을 제공한다.

-----

### **벌크 연산 사용법**

벌크 연산의 문법은 SQL과 거의 동일하다. `executeUpdate()` 메서드를 사용하여 실행하며, 반환 값은 연산의 영향을 받은 로우(row)의 개수다.

**벌크 UPDATE**

```jpql
-- 모든 상품의 가격을 10% 인상
UPDATE Product p SET p.price = p.price * 1.1 WHERE p.stockAmount < 10
```

**벌크 DELETE**

```jpql
-- 30세 미만의 모든 회원 삭제
DELETE FROM Member m WHERE m.age < 30
```

**실행 코드**

```kotlin
@Test
@Transactional
fun `벌크 연산 테스트`() {
    entityManager.persist(Member(username = "User1", age = 25))
    entityManager.persist(Member(username = "User2", age = 35))

    val jpql = "DELETE FROM Member m WHERE m.age < 30"

    // executeUpdate()는 영향을 받은 row의 수를 반환한다.
    val affectedRows = entityManager.createQuery(jpql).executeUpdate()

    assertThat(affectedRows).isEqualTo(1)
}
```

### **가장 중요한 주의점: 영속성 컨텍스트와의 불일치**

벌크 연산은 엄청난 성능상의 이점을 제공하지만, 매우 위험한 부작용을 가지고 있다. 바로 **영속성 컨텍스트를 완전히 무시하고 데이터베이스에 직접 쿼리를 실행한다**는 점이다.

이것이 왜 위험할까? 다음 시나리오를 보자.

1.  `age = 25`인 `memberA`를 조회하여 영속성 컨텍스트의 1차 캐시에 로드했다. 현재 메모리에는 `memberA`의 나이가 25로 캐싱되어 있다.
2.  벌크 연산 `UPDATE Member m SET m.age = m.age + 1` 을 실행했다. 데이터베이스의 `memberA`의 나이는 이제 26이 되었다.
3.  하지만 영속성 컨텍스트는 이 사실을 전혀 알지 못한다. 1차 캐시에 있는 `memberA` 객체의 `age` 필드는 여전히 25다.
4.  이 상태에서 `entityManager.find(Member.class, memberA.id)`를 다시 호출하면, JPA는 DB를 조회하지 않고 1차 캐시에서 `memberA`를 반환한다. 애플리케이션은 이 회원의 나이를 25라고 믿게 된다.

이처럼 **벌크 연산은 메모리(영속성 컨텍스트)와 데이터베이스 간의 데이터 불일치를 유발**하여 심각한 문제를 일으킬 수 있다.

-----

### **해결책: 벌크 연산 후에는 반드시 영속성 컨텍스트를 초기화하라**

이 문제를 해결하는 가장 확실하고 간단한 방법은 **벌크 연산을 수행한 직후, 즉시 영속성 컨텍스트를 초기화(`clear()`)하는 것**이다.

```kotlin
// ... 벌크 연산 실행
val affectedRows = entityManager.createQuery(jpql).executeUpdate()

// 매우 중요! 영속성 컨텍스트를 초기화하여 데이터 불일치를 방지한다.
entityManager.clear()

// 이제 다시 조회하면 DB에서 변경된 최신 데이터를 읽어온다.
val foundMember = entityManager.find(Member.class, memberB.id) // age=36
```

벌크 연산 후 `clear()`를 호출하면, 1차 캐시가 비워지므로 이후의 모든 조회는 데이터베이스에서 직접 데이터를 다시 읽어오게 된다. 이를 통해 데이터 정합성을 유지할 수 있다.

> **결론: 벌크 연산은 강력하지만 위험하다.**
> 대규모 데이터 변경 시 압도적인 성능을 제공하지만, 영속성 컨텍스트를 우회한다는 사실을 절대 잊어서는 안 된다. **`executeUpdate()` 호출 후에는 `clear()`를 호출하는 것을 하나의 규칙처럼 사용해야 한다.**
>
> -----

이것으로 JPQL의 모든 고급 기능을 마스터했다. 우리는 서브쿼리의 한계와 우회 전략부터 시작하여, 다형성 쿼리, 내장 및 사용자 정의 함수, 그리고 벌크 연산의 강력함과 위험성까지 모두 경험했다.

하지만 아무리 강력한 JPQL이라도 모든 것을 해결해 주지는 않는다. 동적으로 쿼리를 생성해야 하거나, JPQL의 표준 스펙을 벗어나는 복잡한 쿼리가 필요한 순간은 반드시 찾아온다. 다음 9장에서는 바로 이런 상황을 위해 준비된 JPA의 또 다른 무기, **Criteria API**와 최후의 보루인 **네이티브 SQL**에 대해 알아볼 것이다.