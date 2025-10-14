## 04\. 결과 매핑: `@SqlResultSetMapping`을 이용한 정교한 DTO 변환

네이티브 SQL로 여러 값을 조회했을 때, `Object[]` 배열로 결과를 받아 수동으로 DTO에 매핑하는 작업은 번거롭고 오류가 발생하기 쉽다. JPQL의 `new` 명령어처럼, 네이티브 SQL의 결과도 DTO로 직접 변환할 수는 없을까?

JPA는 이 문제를 해결하기 위해 \*\*`@SqlResultSetMapping`\*\*이라는 매우 강력하고 정교한 결과 매핑 기능을 제공한다. 이 어노테이션을 사용하면, 어떤 복잡한 SQL 결과셋이라도 원하는 DTO나 엔티티의 조합으로 변환하는 규칙을 미리 정의해 둘 수 있다.

-----

### **`@SqlResultSetMapping` 사용법**

`@SqlResultSetMapping`은 보통 엔티티 클래스 최상단에 정의하며, 여러 개의 매핑 규칙을 정의할 수 있다. DTO로 변환할 때는 `@ConstructorResult`를 함께 사용하는 것이 가장 효과적이다.

**1단계: 결과를 담을 DTO 정의**
먼저, 네이티브 SQL의 조회 결과를 담을 DTO 클래스를 만든다. 이 DTO는 **결과 셋의 컬럼 순서와 타입이 일치하는 생성자**를 가지고 있어야 한다.

**MemberWithTeamNameDto.kt**

```kotlin
data class MemberWithTeamNameDto(
    val username: String,
    val teamName: String
)
```

**2단계: `@SqlResultSetMapping` 정의**
아무 엔티티 클래스(보통 관련 있는 엔티티) 위에 매핑 규칙을 정의한다.

**Member.kt**

```kotlin
@Entity
@SqlResultSetMapping(
    name = "MemberWithTeamNameMapping", // 매핑 규칙에 고유한 이름을 부여한다.
    classes = [
        ConstructorResult(
            targetClass = MemberWithTeamNameDto::class, // 결과를 매핑할 DTO 클래스
            columns = [
                // SQL 결과 셋의 컬럼 순서대로 DTO 생성자에 매핑한다.
                ColumnResult(name = "username", type = String::class),
                ColumnResult(name = "teamName", type = String::class)
            ]
        )
    ]
)
class Member(
    // ...
)
```

  * `@SqlResultSetMapping(name = ...)`: 매핑 규칙의 이름을 정의한다. 이 이름은 나중에 쿼리를 실행할 때 사용된다.
  * `@ConstructorResult(targetClass = ...)`: 결과를 DTO의 생성자를 통해 매핑하겠다고 선언한다.
  * `@ColumnResult(name = ..., type = ...)`: SQL 결과 셋의 컬럼 이름(또는 별칭)과 타입을 지정한다. 여기에 나열된 순서가 `targetClass` 생성자의 파라미터 순서와 일치해야 한다.

**3단계: 네이티브 쿼리 실행 시 매핑 이름 사용**
이제 `entityManager.createNativeQuery()`를 호출할 때, 두 번째 인자로 엔티티 클래스 대신 **`@SqlResultSetMapping`에서 정의한 이름**을 넘겨주면 된다.

```kotlin
@Test
@Transactional
fun `SqlResultSetMapping 테스트`() {
    val teamA = Team(name = "Team A")
    entityManager.persist(teamA)
    entityManager.persist(Member(username = "Kim", team = teamA))

    // SQL 쿼리에서는 컬럼에 별칭(alias)을 부여하여 @ColumnResult의 name과 맞추는 것이 좋다.
    val sql = """
        SELECT m.USERNAME as username, t.NAME as teamName
        FROM MEMBER m
        JOIN TEAM t ON m.TEAM_ID = t.ID
    """

    // 쿼리 실행 시, 클래스 이름 대신 매핑의 이름을 사용한다.
    val query = entityManager.createNativeQuery(sql, "MemberWithTeamNameMapping")

    val resultList = query.resultList as List<MemberWithTeamNameDto>

    assertThat(resultList).hasSize(1)
    assertThat(resultList[0].username).isEqualTo("Kim")
    assertThat(resultList[0].teamName).isEqualTo("Team A")
}
```

결과를 보면, `Object[]`를 다루는 번거로운 과정 없이 네이티브 SQL의 결과가 `MemberWithTeamNameDto` 객체의 리스트로 완벽하게 변환된 것을 확인할 수 있다.

-----

`@SqlResultSetMapping`은 다소 설정이 복잡해 보이지만, 한 번 정의해두면 재사용이 가능하고 복잡한 네이티브 쿼리의 결과를 매우 깔끔하고 타입-안전하게 처리할 수 있게 해준다. 복잡한 리포팅 쿼리나 통계 쿼리처럼 엔티티로 직접 매핑하기 어려운 결과를 다룰 때 이 기능은 매우 강력한 무기가 된다.

이제 JPQL과 네이티브 SQL을 넘어서, 데이터베이스에 미리 정의된 로직의 묶음인 '스토어드 프로시저'를 호출하는 방법을 다음 절에서 알아보자.