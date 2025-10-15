## 04. Spring Data JPA 쿼리 메서드 키워드 목록

스프링 데이터 JPA의 쿼리 메서드는 정해진 키워드들을 조합하여 메서드 이름을 만드는 '규칙 기반' 기능이다. 이 절은 쿼리 메서드를 작성할 때 사용할 수 있는 주요 키워드들을 정리한 빠른 참조 가이드다.

---

### **1. 주요 접두사 (Query Prefixes)**

메서드가 어떤 종류의 작업을 수행할지 결정한다.

| 접두사 | 설명 | 예시 |
| :--- | :--- | :--- |
| `find...By` | 데이터를 조회한다. 가장 일반적으로 사용된다. | `findByUsername(String username)` |
| `read...By` | `find...By`와 기능적으로 동일하다. | `readByUsername(String username)` |
| `get...By` | `find...By`와 기능적으로 동일하다. | `getByUsername(String username)` |
| `query...By`| `find...By`와 기능적으로 동일하다. | `queryByUsername(String username)`|
| `count...By`| 조회 결과의 개수(`Long`)를 반환한다. | `countByTeam(Team team)` |
| `exists...By`| 조회 결과의 존재 여부(`boolean`)를 반환한다. | `existsByUsername(String username)`|
| `delete...By`| 조회된 엔티티를 삭제한다. `@Transactional`이 필요하다. | `deleteByUsername(String username)` |
| `remove...By`| `delete...By`와 기능적으로 동일하다. | `removeByUsername(String username)`|

---

### **2. 조건 키워드 (Conditional Keywords)**

`By` 뒤에 엔티티의 필드명과 조건 키워드를 조합하여 `WHERE` 절을 구성한다.

| 키워드 | 설명 | JPQL 변환 예시 | 메서드명 예시 |
| :--- | :--- | :--- | :--- |
| `And` | 여러 조건을 AND로 연결한다. | `... WHERE x = ? AND y = ?` | `findByUsernameAndAge(...)` |
| `Or` | 여러 조건을 OR로 연결한다. | `... WHERE x = ? OR y = ?` | `findByUsernameOrEmail(...)` |
| `Is`, `Equals`| 동등 비교. 보통 생략한다. (`findByUsername(...)`) | `... WHERE username = ?` | `findByUsernameIs(...)` |
| `Not` | 조건의 결과를 부정한다. | `... WHERE username <> ?` | `findByUsernameNot(...)` |
| `Between` | 두 값 사이의 범위를 조건으로 한다. (양 끝 값 포함) | `... WHERE age BETWEEN ? AND ?` | `findByAgeBetween(20, 30)` |
| `LessThan` | 보다 작음 (`<`) | `... WHERE age < ?` | `findByAgeLessThan(20)` |
| `LessThanEqual`| 보다 작거나 같음 (`<=`) | `... WHERE age <= ?` | `findByAgeLessThanEqual(20)`|
| `GreaterThan` | 보다 큼 (`>`) | `... WHERE age > ?` | `findByAgeGreaterThan(20)` |
| `GreaterThanEqual`| 보다 크거나 같음 (`>=`) | `... WHERE age >= ?` | `findByAgeGreaterThanEqual(20)`|
| `IsNull`, `IsNotNull`| `NULL` 또는 `NOT NULL`을 검사한다. | `... WHERE team IS NULL` | `findByTeamIsNull()` |
| `True`, `False`| `boolean` 필드가 `true` 또는 `false`인지 검사한다. | `... WHERE active = true` | `findByActiveTrue()` |
| `In`, `NotIn` | 컬렉션에 포함되거나 포함되지 않는지를 검사한다. | `... WHERE age IN (?, ?, ?)` | `findByAgeIn(List<Int> ages)` |
| `Like`, `NotLike`| 문자열 패턴 매칭 (`%`, `_` 와일드카드 사용) | `... WHERE username LIKE ?` | `findByUsernameLike("%kim")`|
| `StartingWith`| `LIKE 'kim%'` 와 동일 | `... WHERE username LIKE ?` | `findByUsernameStartingWith("kim")`|
| `EndingWith` | `LIKE '%kim'` 와 동일 | `... WHERE username LIKE ?` | `findByUsernameEndingWith("kim")`|
| `Containing` | `LIKE '%kim%'` 와 동일 | `... WHERE username LIKE ?` | `findByUsernameContaining("kim")`|

---

### **3. 정렬 및 페이징**

* **`OrderBy`**
    * **역할**: 메서드 이름에 직접 정렬 조건을 포함시킨다.
    * **예시**: `findByAgeGreaterThanOrderByUsernameDesc(int age)`
    * **변환**: `... WHERE age > ? ORDER BY username DESC`

* **`Sort` 및 `Pageable` 파라미터**
    * **역할**: 메서드 이름에 `OrderBy`를 사용하는 것보다, `Sort`나 `Pageable` 타입의 파라미터를 추가하여 동적으로 정렬 및 페이징 조건을 전달하는 것이 훨씬 더 유연하고 실용적이다.
    * **예시**:
        * `findByAge(int age, Sort sort)`
        * `findByTeam(Team team, Pageable pageable)`

쿼리 메서드는 매우 편리하지만, 조건이 3~4개를 넘어가면 메서드 이름이 지나치게 길어져 가독성을 해칠 수 있다. 이럴 때는 `@Query` 어노테이션이나 Querydsl을 사용하는 것이 더 나은 선택이다.