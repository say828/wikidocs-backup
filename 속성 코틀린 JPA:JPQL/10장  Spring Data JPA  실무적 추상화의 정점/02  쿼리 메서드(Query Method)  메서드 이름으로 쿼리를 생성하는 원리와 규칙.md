## 02\. 쿼리 메서드(Query Method): 메서드 이름으로 쿼리를 생성하는 원리와 규칙

`JpaRepository`가 제공하는 기본 CRUD 메서드만으로는 실무의 다양한 조회 요구사항을 모두 만족시킬 수 없다. "특정 이름으로 회원을 조회"하거나, "특정 나이보다 많으면서 이름으로 정렬된 회원 목록을 조회"하는 기능은 어떻게 만들어야 할까?

순수 JPA를 사용했다면 JPQL을 직접 작성해야 했을 것이다. 하지만 스프링 데이터 JPA는 훨씬 더 경이로운 방법을 제공한다. 바로 **메서드 이름 그 자체를 분석하여 JPQL 쿼리를 자동으로 생성**해주는, 일명 **쿼리 메서드(Query Method)** 기능이다.

-----

### **쿼리 메서드의 마법**

개발자는 그저 정해진 규칙에 따라 리포지토리 인터페이스에 **메서드를 선언하기만 하면 된다.** 구현은 전혀 필요 없다.

**MemberRepository.kt**

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {

    // username으로 회원을 조회하는 쿼리 메서드
    fun findByUsername(username: String): Member?

    // age보다 나이가 많은 회원을 username으로 내림차순 정렬하여 조회
    fun findByAgeGreaterThanOrderByUsernameDesc(age: Int): List<Member>
}
```

이게 전부다. 애플리케이션이 시작될 때, 스프링 데이터 JPA는 이 메서드 이름들을 파싱하여 다음과 같은 JPQL을 자동으로 생성하고 구현해준다.

  * `findByUsername` -\> `SELECT m FROM Member m WHERE m.username = :username`
  * `findByAgeGreaterThanOrderByUsernameDesc` -\> `SELECT m FROM Member m WHERE m.age > :age ORDER BY m.username DESC`

개발자는 이제 이 메서드를 서비스 계층에서 호출하기만 하면 된다. JPQL 문자열의 오타 걱정 없이, 메서드 시그니처만으로 어떤 쿼리가 실행될지 명확하게 알 수 있다.

-----

### **쿼리 메서드 생성 규칙**

스프링 데이터 JPA가 메서드 이름을 쿼리로 변환하는 데는 일정한 규칙(Convention)이 있다.

**`find...By...`, `read...By...`, `get...By...`, `query...By...`, `count...By...`, `exists...By...`** 와 같은 정해진 접두어로 시작하고, 그 뒤에 엔티티의 필드 이름을 조합하여 조건을 만든다.

  * **`And`, `Or`**: 여러 조건을 조합할 때 사용한다.

      * `findByUsernameAndAge(String username, int age)`

  * **비교 연산자**:

      * `Is`, `Equals` (생략 가능): `findByUsername`은 `findByUsernameIs`와 같다.
      * `GreaterThan`, `GreaterThanEqual`: `findByAgeGreaterThan(20)`
      * `LessThan`, `LessThanEqual`
      * `Between`: `findByAgeBetween(20, 30)`
      * `IsNull`, `IsNotNull`: `findByTeamIsNull()`
      * `True`, `False`: `findByIsActiveTrue()`

  * **문자열 연산자**:

      * `Like`, `NotLike`: `findByUsernameLike("%Kim%")`
      * `StartingWith`, `EndingWith`, `Containing`: `Like`의 축약형. `findByUsernameStartingWith("Kim")` -\> `WHERE username LIKE 'Kim%'`

  * **정렬**: `OrderBy` 키워드 뒤에 필드 이름과 정렬 방향(`Asc`, `Desc`)을 붙인다.

      * `findAllByOrderByAgeDescUsernameAsc()`

  * **반환 타입**: 스프링 데이터 JPA는 메서드의 반환 타입에 맞춰 유연하게 결과를 반환한다.

      * `Member?` (또는 `Optional<Member>`): 단 건 조회. 결과가 없으면 `null` 또는 `Empty`를 반환한다.
      * `List<Member>`: 여러 건 조회. 결과가 없으면 빈 리스트를 반환한다.
      * `Long` (또는 `int`): `countBy...` 메서드의 경우, 조건에 맞는 데이터의 개수를 반환한다.

쿼리 메서드는 단순한 조회 기능의 생산성을 극적으로 끌어올려 주는 스프링 데이터 JPA의 핵심 기능이다. 하지만 메서드 이름이 너무 길어지면 가독성이 떨어지고, 복잡한 조인이나 DTO 변환 같은 작업은 처리하기 어렵다는 명확한 한계가 있다.

그렇다면 메서드 이름만으로 표현하기 힘든 복잡한 쿼리는 어떻게 작성해야 할까? 다음 절에서는 리포지토리 인터페이스에 직접 JPQL을 바인딩하는 `@Query` 어노테이션에 대해 알아본다.