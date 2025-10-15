## 04\. `new` 명령어를 이용한 DTO 직접 변환: 실무적 접근과 성능

이전 절에서 스칼라 타입으로 여러 필드를 조회했을 때, `Object[]` 배열로 결과를 받아 타입 캐스팅을 반복해야 하는 불편함을 겪었다. 이런 방식은 코드를 지저분하게 만들고, 런타임에 `ClassCastException`을 유발할 수 있는 잠재적인 위험을 안고 있다.

실무에서는 조회 결과를 엔티티가 아닌, 화면 렌더링이나 API 응답에 최적화된 별도의 \*\*DTO(Data Transfer Object)\*\*로 받아 사용하는 경우가 대부분이다. JPQL은 **`new` 명령어**를 통해 이 과정을 매우 깔끔하고 효율적으로 처리할 수 있는 방법을 제공한다.

-----

### **JPQL `new` 명령어 사용법**

`new` 명령어는 JPQL 쿼리 결과로 즉석에서 DTO 객체를 생성하여 반환하도록 지시하는 기능이다.

**문법**: `SELECT new 패키지명을 포함한 클래스명(프로젝션 필드1, 필드2, ...)`

**사용 규칙**은 다음과 같다.

1.  `new` 키워드 뒤에는 반드시 \*\*패키지명을 포함한 전체 클래스 경로(Fully Qualified Class Name)\*\*를 적어야 한다.
2.  지정한 DTO 클래스에는 JPQL의 `SELECT` 절에 나열된 필드들의 **타입과 순서가 정확히 일치하는 생성자**가 반드시 존재해야 한다.

-----

### **코드로 보는 DTO 변환**

먼저, 조회 결과를 담을 DTO 클래스를 정의하자.

**MemberDto.kt**

```kotlin
package com.masterclass.jpamaster.dto

// JPQL 조회 결과를 담을 DTO.
// username(String), age(Int) 순서의 생성자를 가지고 있다.
data class MemberDto(
    val username: String,
    val age: Int
)
```

이제 이 `MemberDto`를 사용하여 JPQL 쿼리를 작성하고 실행해 보자.

**JPQL 실행 코드**

```kotlin
@Test
@Transactional
fun `new 명령어로 DTO 직접 변환 테스트`() {
    // 테스트 데이터 준비
    entityManager.persist(Member(username = "Kim", age = 30))
    entityManager.persist(Member(username = "Park", age = 25))

    // 1. DTO 클래스의 전체 경로를 사용하여 JPQL 작성
    val jpql = "SELECT new com.masterclass.jpamaster.dto.MemberDto(m.username, m.age) " +
               "FROM Member m"

    // 2. 반환 타입을 DTO 클래스로 지정하여 쿼리 생성
    val query = entityManager.createQuery(jpql, MemberDto::class.java)

    // 3. 쿼리 실행 -> 이제 결과는 List<MemberDto> 이다!
    val resultList: List<MemberDto> = query.resultList

    // 4. 결과 확인
    assertThat(resultList).hasSize(2)
    assertThat(resultList[0].username).isEqualTo("Kim")
    assertThat(resultList[0].age).isEqualTo(30)
}
```

결과를 보라. `Object[]`를 다룰 때와는 비교할 수 없을 정도로 코드가 깔끔하고 타입-안전(type-safe)해졌다. JPA는 JPQL을 실행하여 얻은 `username`과 `age` 값을 `MemberDto`의 생성자에 순서대로 전달하여 객체를 만들고, 우리는 그 결과를 `List<MemberDto>`로 직접 받았다. 더 이상의 번거로운 타입 캐스팅은 필요 없다.

이 방식은 단순히 코드의 편의성만 높이는 것이 아니다. `SELECT` 절에 꼭 필요한 필드만 지정하므로 데이터베이스에서 애플리케이션으로 전송되는 데이터의 양을 최소화하여 성능에도 긍정적인 영향을 미친다.

> **결론: DTO 조회가 필요할 땐 `new` 명령어를 사용하라.**
> 엔티티 전체가 아닌, 특정 필드들만 조회하여 DTO로 변환하고 싶을 때 `new` 명령어는 가장 실용적이고 효율적인 해결책이다.

이제 우리는 원하는 데이터를 원하는 형태로 조회하는 법을 배웠다. 하지만 지금까지의 `WHERE` 절은 `"WHERE m.age > 20"` 처럼 조건 값이 하드코딩되어 있었다. 실제 애플리케이션에서는 이 값이 동적으로 변해야 한다. 다음 절에서는 이 동적 파라미터를 안전하게 처리하는 방법에 대해 알아보자.