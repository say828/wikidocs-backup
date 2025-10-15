## 03\. JPQL 내장 함수: `COALESCE`, `NULLIF`, `CONCAT`, `SIZE` 등

JPQL은 단순히 엔티티를 조회하고 필터링하는 것을 넘어, 쿼리 내에서 데이터를 조작하고 표현을 풍부하게 만들어주는 다양한 \*\*표준 내장 함수(Built-in Functions)\*\*를 제공한다. 이 함수들은 대부분의 데이터베이스에서 공통으로 지원하는 표준 SQL 함수와 유사하며, JPQL을 통해 사용하면 데이터베이스 방언(Dialect)에 상관없이 이식성 있는 쿼리를 작성할 수 있다는 큰 장점이 있다.

이번 절에서는 실무에서 유용하게 사용되는 몇 가지 핵심 내장 함수들을 소개한다.

-----

### **주요 내장 함수**

**1. 문자열 함수 (String Functions)**

  * `CONCAT('a', 'b')`: 두 문자열을 합친다. (예: `CONCAT(m.username, ':', m.age)`)
  * `SUBSTRING(string, start, length)`: 문자열을 특정 위치에서부터 지정된 길이만큼 잘라낸다.
  * `TRIM(string)`: 문자열의 양쪽 공백을 제거한다.
  * `LOWER(string)`, `UPPER(string)`: 문자열을 소문자 또는 대문자로 변환한다.
  * `LOCATE(find, source, start)`: 특정 문자열(`find`)이 원본 문자열(`source`)에서 나타나는 위치를 찾는다.

**2. 수학 함수 (Numeric Functions)**

  * `ABS(number)`: 숫자의 절댓값을 반환한다.
  * `SQRT(number)`: 숫자의 제곱근을 반환한다.
  * `MOD(number1, number2)`: `number1`을 `number2`로 나눈 나머지를 구한다.

**3. 컬렉션 함수 (Collection Functions)**

  * `SIZE(collection)`: 컬렉션의 크기를 반환한다. **매우 유용하게 사용된다.**
    ```jpql
    -- 예제: 회원을 2명 이상 보유한 팀 조회
    SELECT t FROM Team t WHERE SIZE(t.members) >= 2
    ```
    이 JPQL은 `LEFT JOIN`과 `GROUP BY`, `HAVING COUNT(...)`를 사용하는 SQL로 변환되어 실행된다. `SIZE` 함수 덕분에 훨씬 직관적인 쿼리 작성이 가능하다.

**4. 표준 CASE 표현식**
JPQL은 SQL의 `CASE` 문과 거의 동일한 문법을 지원하여 쿼리 내에서 조건부 로직을 구현할 수 있게 해준다. `COALESCE`와 `NULLIF`는 `CASE` 문의 특별한 형태라고 볼 수 있다.

  * **`COALESCE(value1, value2, ...)`**: 인자들 중에서 `NULL`이 아닌 첫 번째 값을 반환한다.
    ```jpql
    -- 예제: 회원의 이름이 없으면 '이름 없는 회원'을 대신 출력
    SELECT COALESCE(m.username, '이름 없는 회원') FROM Member m
    ```
  * **`NULLIF(value1, value2)`**: 두 값이 같으면 `NULL`을, 다르면 첫 번째 값(`value1`)을 반환한다.
    ```jpql
    -- 예제: 회원의 이름이 '관리자'이면 NULL로 처리하고 싶을 때
    SELECT NULLIF(m.username, '관리자') FROM Member m
    ```

-----

### **코드 예시**

```kotlin
@Test
@Transactional
fun `내장 함수 테스트`() {
    val member = Member(username = "   Master Kim   ", age = 30)
    entityManager.persist(member)

    // CONCAT, TRIM, UPPER, SIZE 등 다양한 함수 사용
    val jpql = """
        SELECT CONCAT('User: ', UPPER(TRIM(m.username))), 
               'Age: ', 
               m.age 
        FROM Member m
    """

    val result = entityManager.createQuery(jpql).singleResult as Array<Any>

    assertThat(result[0]).isEqualTo("User: MASTER KIM")
    assertThat(result[2]).isEqualTo(30)
}
```

이처럼 내장 함수를 활용하면 애플리케이션 레벨에서 처리해야 할 데이터 가공 로직을 쿼리 단계로 옮겨, 더 효율적이고 간결한 코드를 작성할 수 있다.

하지만 만약 우리가 사용하는 데이터베이스가 JPQL 표준에는 없는 특별하고 유용한 함수를 제공한다면 어떻게 해야 할까? 다음 절에서는 이 문제를 해결하는 사용자 정의 함수에 대해 알아본다.