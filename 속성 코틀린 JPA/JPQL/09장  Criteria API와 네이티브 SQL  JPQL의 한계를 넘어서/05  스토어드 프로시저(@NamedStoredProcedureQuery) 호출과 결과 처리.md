## 05\. 스토어드 프로시저(@NamedStoredProcedureQuery) 호출과 결과 처리

JPA의 쿼리 기술 여정, 그 마지막 주제는 \*\*스토어드 프로시저(Stored Procedure)\*\*다. 스토어드 프로시저는 데이터베이스에 미리 정의해 둔 SQL 쿼리의 묶음으로, 복잡한 비즈니스 로직을 데이터베이스 단에서 캡슐화하여 처리하고자 할 때 사용된다.

최근에는 비즈니스 로직을 애플리케이션 계층에 두는 것을 선호하는 추세지만, 레거시 시스템과 연동하거나, 데이터베이스 중심적인 아키텍처를 가진 조직에서는 여전히 스토어드 프로시저가 활발히 사용된다. JPA는 이러한 환경을 위해 스토어드 프로시저를 호출하고 결과를 매핑하는 표준화된 방법을 제공한다.

-----

### **`@NamedStoredProcedureQuery`를 이용한 프로시저 호출**

가장 표준적인 방법은 `@NamedStoredProcedureQuery` 어노테이션을 사용하여 호출할 프로시저를 미리 등록해 두는 것이다.

**1단계: 데이터베이스에 스토어드 프로시저 생성**
먼저, 테스트를 위해 H2 데이터베이스에 간단한 프로시저를 하나 만들자. 회원 ID를 입력받아 그 회원의 이름을 반환하는 프로시저다.

```sql
-- H2 데이터베이스용 스토어드 프로시저
CREATE OR REPLACE PROCEDURE proc_find_username_by_id(
    IN member_id BIGINT,
    OUT out_username VARCHAR(255)
) AS
BEGIN
    SELECT username INTO out_username FROM member WHERE id = member_id;
END;
```

**2단계: `@NamedStoredProcedureQuery` 정의**
엔티티 클래스 위에 호출할 프로시저의 명세를 정의한다.

**Member.kt**

```kotlin
@Entity
@NamedStoredProcedureQuery(
    name = "Member.findUsernameById", // 프로시저 쿼리에 부여할 고유한 이름
    procedureName = "proc_find_username_by_id", // DB에 정의된 실제 프로시저 이름
    parameters = [
        // IN: 입력 파라미터
        StoredProcedureParameter(mode = ParameterMode.IN, name = "member_id", type = Long::class),
        // OUT: 출력 파라미터
        StoredProcedureParameter(mode = ParameterMode.OUT, name = "out_username", type = String::class)
    ]
)
class Member(
    // ...
)
```

  * `name`: JPA에서 이 프로시저 호출을 식별하기 위한 이름.
  * `procedureName`: 데이터베이스에 실제 생성된 프로시저의 이름.
  * `parameters`: 프로시저가 받을 파라미터를 `@StoredProcedureParameter`로 정의한다. `mode`(`IN`, `OUT`, `INOUT`, `REF_CURSOR`)와 이름, 타입을 명시한다.

**3단계: 프로시저 실행 코드**
이제 `EntityManager`를 통해 등록된 이름으로 프로시저 쿼리를 얻어와 실행할 수 있다.

```kotlin
@Test
@Transactional
fun `스토어드 프로시저 호출 테스트`() {
    val member = Member(username = "Kim", age = 30)
    entityManager.persist(member)

    // 1. 이름으로 StoredProcedureQuery 객체 생성
    val query = entityManager.createNamedStoredProcedureQuery("Member.findUsernameById")

    // 2. IN 파라미터 값 설정
    query.setParameter("member_id", member.id)

    // 3. 프로시저 실행
    query.execute()

    // 4. OUT 파라미터 값 조회
    val outUsername = query.getOutputParameterValue("out_username") as String

    assertThat(outUsername).isEqualTo("Kim")
}
```

`createNamedStoredProcedureQuery()`로 쿼리를 생성하고, `setParameter()`로 입력 값을 설정한 뒤 `execute()`로 실행한다. 결과는 `getOutputParameterValue()`를 통해 받아올 수 있다.

-----

JPA의 스토어드 프로시저 지원 기능은 데이터베이스에 깊이 의존하는 로직을 객체지향적인 코드와 통합할 수 있는 중요한 통로 역할을 한다. 비록 현대적인 애플리케이션 설계에서는 그 사용 빈도가 줄어들고 있지만, 기존 시스템과의 연동이나 특정 요구사항 해결을 위해 알아두어야 할 가치 있는 기술이다.

-----

이것으로 JPQL의 한계를 넘어서는 Criteria, 네이티브 SQL, 그리고 스토어드 프로시저까지, JPA가 제공하는 모든 쿼리 기술의 스펙트럼을 탐험했다. 우리는 이제 어떤 복잡한 조회 요구사항이 닥쳐도 해결할 수 있는 다양한 무기를 갖추게 되었다.

하지만 지금까지의 과정은 다소 '날 것'에 가까웠다. `EntityManager`를 직접 호출하고, 쿼리 문자열을 작성하는 등 반복적인 작업이 많았다. 현대적인 애플리케이션 개발은 더 높은 생산성을 요구한다. 다음 10장에서는 이 모든 저수준의 작업을 놀랍도록 간단하게 추상화하여, 개발자가 오직 비즈니스 로직에만 집중할 수 있게 해주는 마법 같은 기술, \*\*스프링 데이터 JPA(Spring Data JPA)\*\*의 세계로 떠나볼 것이다.