## 03\. @Query 어노테이션: JPQL과 네이티브 쿼리를 리포지토리에 바인딩

쿼리 메서드는 단순한 조회에서는 놀라운 생산성을 보여주지만, 그 한계는 명확하다. 메서드 이름이 너무 길어져 가독성을 해치거나, 여러 테이블을 조인하거나, DTO로 직접 결과를 반환받는 등 복잡한 로직을 표현하기에는 역부족이다.

이처럼 **메서드 이름만으로는 해결할 수 없는 복잡한 쿼리**를 위해 스프링 데이터 JPA는 **`@Query` 어노테이션**을 제공한다. `@Query`를 사용하면 리포지토리 인터페이스의 메서드에 JPQL이나 순수 SQL(네이티브 쿼리)을 직접 매핑할 수 있다. 이는 쿼리 메서드의 편리함과 JPQL의 강력함을 결합한, 실무에서 가장 널리 사용되는 기능 중 하나다.

-----

### **메서드에 JPQL 직접 작성하기**

`@Query` 어노테이션을 메서드 위에 붙이고, 그 안에 실행하고 싶은 JPQL을 문자열로 작성하면 된다.

**MemberRepository.kt**

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {

    // 복잡한 비즈니스 로직이 담긴 JPQL을 직접 작성
    @Query("SELECT m FROM Member m WHERE m.username = :username AND m.age = :age")
    fun findUser(@Param("username") username: String, @Param("age") age: Int): Member?
}
```

  * **`@Query(...)`**: 이 메서드는 자동으로 생성되는 것이 아니라, 어노테이션 안에 있는 JPQL을 실행하도록 지정한다.
  * **`:username`, `:age`**: JPQL의 이름 기반 파라미터다.
  * **`@Param("username")`**: 메서드의 파라미터(`username`)를 JPQL의 이름 기반 파라미터(`:username`)에 바인딩한다.

이 방식은 메서드 이름은 간결하게 유지하면서, 쿼리의 내용은 JPQL로 자유롭게 표현할 수 있다는 큰 장점이 있다. 정적인 쿼리(항상 같은 형태로 실행되는 쿼리)에 매우 효과적이다.

-----

### **DTO로 직접 조회하는 기능**

`@Query`의 진정한 힘은 JPQL의 `new` 명령어와 결합될 때 발휘된다. 복잡한 조인 결과나 통계 데이터를 엔티티가 아닌 DTO로 직접 조회하는 실무의 가장 흔한 요구사항을 매우 깔끔하게 해결할 수 있다.

**MemberDto.kt**

```kotlin
data class MemberDto(
    val username: String,
    val teamName: String
)
```

**MemberRepository.kt**

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {

    // Member와 Team을 조인하여 MemberDto로 직접 반환
    @Query("SELECT new com.masterclass.jpamaster.dto.MemberDto(m.username, t.name) " +
           "FROM Member m JOIN m.team t")
    fun findMemberDto(): List<MemberDto>
}
```

`@Query` 안에 `new` 명령어를 사용한 JPQL을 작성하는 것만으로, 스프링 데이터 JPA는 조인 결과를 `MemberDto` 객체의 리스트로 완벽하게 변환하여 반환해준다. 더 이상 `EntityManager`를 직접 다루거나 `Object[]`를 파싱할 필요가 없다.

-----

### **네이티브 쿼리 사용하기**

때로는 JPQL이 지원하지 않는 특정 데이터베이스의 고유한 함수나 문법을 사용해야 할 때가 있다. `@Query`는 `nativeQuery = true` 속성을 통해 순수한 SQL(네이티브 쿼리)도 지원한다.

```kotlin
// 특정 데이터베이스에 종속적인 쿼리 실행
@Query(value = "SELECT * FROM member WHERE username = ?1", nativeQuery = true)
fun findByUsernameNative(username: String): Member?
```

이 기능은 강력하지만, 사용하는 순간 데이터베이스 이식성을 잃게 되므로 신중하게 사용해야 한다.

`@Query` 어노테이션은 쿼리 메서드의 단순함과 JPQL의 유연성 사이에서 완벽한 균형을 제공하는 실무의 핵심 도구다. 하지만 사용자의 입력에 따라 검색 조건이 계속해서 변하는 '동적 쿼리'를 처리하기에는 여전히 불편함이 남는다. 다음 절에서는 이 동적 쿼리 문제를 해결하는 두 가지 방법, 명세(Specification)와 Querydsl에 대해 알아볼 것이다.