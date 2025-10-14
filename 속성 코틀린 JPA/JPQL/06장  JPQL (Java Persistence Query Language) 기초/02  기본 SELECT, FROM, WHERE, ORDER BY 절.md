## 기본 SELECT, FROM, WHERE, ORDER BY 절

JPQL의 기본 문법 구조는 SQL과 거의 동일합니다. SQL에 익숙하다면 10분 만에 JPQL의 기본 쿼리를 작성할 수 있습니다. 차이점은 오직 '대상'뿐입니다. (테이블이 아닌 엔티티)

JPQL의 가장 기본적인 쿼리 구조는 다음과 같습니다.

```jpql
SELECT ...  -- [필수] 무엇을 조회할 것인가? (조회 대상)
FROM ...    -- [필수] 어떤 엔티티에서? (쿼리 대상 엔티티)
WHERE ...   -- [선택] 어떤 조건으로? (필터링 조건)
GROUP BY ...
HAVING ...
ORDER BY ... -- [선택] 어떻게 정렬할 것인가? (정렬 조건)
```

이번 섹션에서는 가장 핵심이 되는 `FROM`, `SELECT`, `WHERE`, `ORDER BY` 절을 엔티티 기준으로 살펴보겠습니다.

-----

### 1\. FROM 절 (가장 중요)

JPQL에서 가장 먼저 이해해야 할 절입니다.

  * **대상**: DB 테이블이 아닌, **`@Entity` 어노테이션이 붙은 클래스 이름**을 사용합니다.
  * **별칭(Alias)**: **반드시 별칭을 사용해야 합니다.** (예: `Member m`)

<!-- end list -->

```jpql
-- Member 엔티티를 m이라는 별칭으로 조회
FROM Member m
```

  * `FROM MEMBER` (테이블명)이 아니라 `FROM Member` (클래스명)입니다.
  * `Member`는 대소문자를 구분합니다. (클래스명과 정확히 일치해야 함)
  * `m`은 이 JPQL 쿼리 내에서 `Member` 엔티티 인스턴스를 가리키는 '변수' 역할을 합니다. 이후 모든 절(`SELECT`, `WHERE` 등)에서 이 별칭을 사용해 엔티티의 필드에 접근합니다.

-----

### 2\. SELECT 절

조회할 대상을 지정합니다. SQL과 달리, JPQL의 `SELECT` 절은 다양한 대상을 조회할 수 있습니다.

#### 타입 1: 엔티티 프로젝션 (가장 기본)

별칭 자체를 조회합니다.

```jpql
SELECT m FROM Member m
```

  * **의미**: "Member 엔티티 객체 \*\*'자체'\*\*를 조회하라."
  * **결과**: `List<Member>` 타입으로 반환됩니다.
  * **[매우 중요]**: 이 쿼리를 통해 반환된 모든 `Member` 객체는 **'영속성 컨텍스트(Persistence Context)'에 의해 '관리'되는 영속(Managed) 상태**가 됩니다. (02장 참고)
      * 즉, 이 쿼리로 조회한 객체의 필드를 변경하면(`member.setUsername(...)`), 트랜잭션 커밋 시 '변경 감지(Dirty Checking)'에 의해 `UPDATE` 쿼리가 자동으로 실행됩니다.

#### 타입 2: 스칼라 프로젝션 (특정 필드)

엔티티의 특정 필드(프로퍼티)를 조회합니다.

```jpql
SELECT m.username, m.age FROM Member m
```

  * **의미**: "Member 엔티티의 'username 필드'와 'age 필드'만 조회하라."
  * **결과**: 특정 타입(`Member`)으로 반환할 수 없습니다. 기본적으로 `List<Object[]>` 타입으로 반환됩니다. (이것을 DTO로 직접 받는 방법은 다음 섹션에서 다룹니다.)

-----

### 3\. WHERE 절

SQL과 마찬가지로 조회할 엔티티를 필터링하는 조건을 지정합니다. 역시 '컬럼명'이 아닌 '필드명(프로퍼티명)'을 사용합니다.

```jpql
-- 이름이 'kim'이고 나이가 20살 이상인 회원
SELECT m 
FROM Member m
WHERE m.username = 'kim' AND m.age >= 20
```

  * `USERNAME = 'kim'` (X) -\> `m.username = 'kim'` (O)
  * 객체 그래프 탐색도 가능합니다. (SQL과의 강력한 차별점)
    ```jpql
    -- 1번 팀에 소속된 모든 회원
    SELECT m 
    FROM Member m
    WHERE m.team.id = 1  -- 👈 m.team을 통해 Team 엔티티의 id 필드에 바로 접근
    ```

-----

### 4\. ORDER BY 절

결과를 정렬하는 기준을 명시합니다. '필드명'을 사용하며 `ASC`(오름차순, 기본값) 또는 `DESC`(내림차순)를 지정할 수 있습니다.

```jpql
-- 나이가 많은 순으로 정렬하고, 나이가 같다면 이름 오름차순으로 정렬
SELECT m
FROM Member m
WHERE m.age >= 20
ORDER BY m.age DESC, m.username ASC
```

-----

### 기본 쿼리 실행 예제 (Java / Kotlin)

`EntityManager`를 사용하여 "나이가 18세 이상인 회원을, 이름 가나다순으로 조회하는" 기본 JPQL 쿼리 예제입니다.

#### Java

```java
import jakarta.persistence.EntityManager;
import jakarta.persistence.TypedQuery;
import java.util.List;

// ...
public List<Member> findMembersOlderThan(EntityManager em, int age) {
    // 1. JPQL 문자열 정의 (엔티티와 필드 기준)
    String jpql = "SELECT m FROM Member m " +
                  "WHERE m.age >= :userAge " +  // 👈 파라미터 바인딩 (다음 섹션 참고)
                  "ORDER BY m.username ASC";

    // 2. 쿼리 생성 (TypedQuery 사용 시 반환 타입 명시)
    TypedQuery<Member> query = em.createQuery(jpql, Member.class);

    // 3. 파라미터 설정
    query.setParameter("userAge", age);

    // 4. 쿼리 실행 및 결과 반환
    List<Member> members = query.getResultList();
    
    // 5. [중요] 이 'members' 리스트 안의 모든 Member 객체는 '영속(Managed)' 상태입니다.
    return members;
}
```

#### Kotlin

```kotlin
import jakarta.persistence.EntityManager
import jakarta.persistence.TypedQuery

// ...
fun findMembersOlderThan(em: EntityManager, age: Int): List<Member> {
    // 1. JPQL 문자열 정의 (Triple-quote 사용)
    val jpql = """
        SELECT m 
        FROM Member m 
        WHERE m.age >= :userAge 
        ORDER BY m.username ASC
    """

    // 2. 쿼리 생성 (::class.java로 KClass를 Java Class로 변환)
    val query: TypedQuery<Member> = em.createQuery(jpql, Member::class.java)

    // 3. 파라미터 설정
    query.setParameter("userAge", age)

    // 4. 쿼리 실행 (프로퍼티 접근)
    val members: List<Member> = query.resultList
    
    // 5. [중요] 이 'members' 리스트 안의 모든 Member 객체는 '영속(Managed)' 상태입니다.
    return members
}
```