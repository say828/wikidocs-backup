## 05\. 파라미터 바인딩: 이름 기준(:name)과 위치 기준(?1) (SQL Injection 방어)

지금까지 작성한 JPQL 쿼리는 `WHERE m.username = 'Kim'` 처럼 검색 조건이 문자열 안에 하드코딩되어 있었다. 실제 애플리케이션에서는 이 값이 사용자의 입력에 따라 동적으로 변해야 한다. 가장 순진한 방법은 쿼리 문자열을 직접 조립하는 것이다.

```kotlin
// 절대 이렇게 사용하면 안 되는 최악의 코드!
val username = "Kim"
val jpql = "SELECT m FROM Member m WHERE m.username = '" + username + "'"
val query = entityManager.createQuery(jpql, Member::class.java)
```

이 방식은 **SQL Injection**이라는 최악의 보안 취약점에 그대로 노출된다. 만약 악의적인 사용자가 `username` 값으로 `' OR '1' = '1` 과 같은 문자열을 입력한다면, `WHERE` 절이 항상 `true`가 되어 모든 회원의 정보가 유출되는 끔찍한 사태가 발생할 수 있다.

이러한 문제를 해결하기 위해 JPA는 \*\*파라미터 바인딩(Parameter Binding)\*\*이라는 안전하고 표준화된 방법을 제공한다. 파라미터 바인딩은 쿼리 문에는 변수가 들어갈 자리를 표시하는 '플레이스홀더(placeholder)'만 남겨두고, 실제 값은 별도의 메서드를 통해 안전하게 주입하는 방식이다. JPA는 이름 기준과 위치 기준, 두 가지 스타일의 파라미터 바인딩을 지원한다.

-----

### **1. 이름 기준 파라미터 (Named Parameters) - ⭐️ 권장**

이름 기준 파라미터는 플레이스홀더를 콜론(`:`) 뒤에 이름을 붙여서 정의하는 방식이다. (예: `:username`, `:age`)

  * **장점**: 파라미터의 순서가 바뀌어도 상관없으며, 이름 자체가 파라미터의 의미를 명확하게 설명해주므로 가독성이 매우 좋다. **실무에서 가장 권장되는 방식이다.**

**코드 예시**

```kotlin
@Test
@Transactional
fun `이름 기준 파라미터 바인딩 테스트`() {
    entityManager.persist(Member(username = "Kim", age = 30))

    val jpql = "SELECT m FROM Member m WHERE m.username = :username"

    val query = entityManager.createQuery(jpql, Member::class.java)
    // ":username" 이라는 이름의 파라미터에 "Kim" 값을 바인딩한다.
    query.setParameter("username", "Kim")

    val result = query.singleResult

    assertThat(result.username).isEqualTo("Kim")
}
```

`setParameter("파라미터이름", 값)` 메서드를 사용하여 이름으로 값을 바인딩한다. 메서드 체이닝(chaining)도 가능하다.

-----

### **2. 위치 기준 파라미터 (Positional Parameters)**

위치 기준 파라미터는 플레이스홀더를 물음표(`?`) 뒤에 숫자를 붙여서 정의하는 방식이다. (예: `?1`, `?2`) **숫자는 1부터 시작한다.**

  * **단점**: 파라미터의 순서가 매우 중요하며, 순서가 바뀌면 쿼리가 오동작하거나 에러가 발생한다. 쿼리가 복잡해지고 파라미터가 많아질수록 유지보수가 매우 어려워진다.

**코드 예시**

```kotlin
@Test
@Transactional
fun `위치 기준 파라미터 바인딩 테스트`() {
    entityManager.persist(Member(username = "Kim", age = 30))

    // ?1: 첫 번째 파라미터를 의미
    val jpql = "SELECT m FROM Member m WHERE m.username = ?1"

    val query = entityManager.createQuery(jpql, Member::class.java)
    // 1번 위치의 파라미터에 "Kim" 값을 바인딩한다.
    query.setParameter(1, "Kim")

    val result = query.singleResult

    assertThat(result.username).isEqualTo("Kim")
}
```

`setParameter(위치, 값)` 메서드를 사용한다. 직관적이지 않고 실수할 여지가 많아, 특별한 경우가 아니라면 이름 기준 파라미터를 사용하는 것이 좋다.

-----

파라미터 바인딩은 선택이 아닌 필수다. 이 방식을 사용하면 JPA가 파라미터로 입력된 값을 단순한 '데이터'로만 인식하도록 처리해주므로, SQL Injection 공격을 원천적으로 차단할 수 있다.

이제 안전하게 쿼리를 작성하는 법까지 배웠다. 하지만 만약 쿼리 결과가 수백만 건이라면 어떻게 해야 할까? 이 모든 데이터를 한 번에 메모리에 올리는 것은 불가능하다. 다음 절에서는 이 문제를 해결하기 위한 페이징 API에 대해 알아보자.