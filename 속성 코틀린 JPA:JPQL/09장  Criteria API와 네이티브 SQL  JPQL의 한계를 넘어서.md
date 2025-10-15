# 09장: Criteria API와 네이티브 SQL: JPQL의 한계를 넘어서

8장에 걸쳐 우리는 JPQL이라는 강력한 언어를 마스터했다. JPQL은 객체지향적인 추상화를 유지하면서 데이터베이스에 독립적인 쿼리를 작성할 수 있게 해주는 훌륭한 도구다. 하지만 JPQL은 그 자체로 완전한 해결책은 아니다. 특히 다음과 같은 상황에서는 JPQL의 한계가 명확히 드러난다.

1.  **동적 쿼리(Dynamic Query)**: 사용자의 검색 조건에 따라 `WHERE` 절이 복잡하게 변해야 하는 경우, JPQL 문자열을 조립하는 것은 매우 번거롭고 오류가 발생하기 쉽다.
2.  **데이터베이스 고유 기능**: 특정 데이터베이스만 지원하는 함수나 SQL 문법(예: Oracle의 계층형 쿼리)을 사용하고 싶을 때, 표준 JPQL로는 해결할 수 없다.

이번 장에서는 바로 이 JPQL의 한계를 뛰어넘기 위한 두 가지 최후의 무기, **Criteria API**와 **네이티브 SQL**에 대해 알아본다. Criteria는 문자열이 아닌 코드로 쿼리를 작성하여 동적 쿼리를 타입-안전(Type-Safe)하게 처리하는 방법을 제공한다. 네이티브 SQL은 JPA의 통제를 벗어나 데이터베이스에 직접 말을 거는, 가장 강력하고 유연한 최후의 수단이다. 이 두 가지를 마스터하면, 여러분은 JPA가 제공하는 쿼리 기술의 모든 스펙트럼을 이해하고 어떤 요구사항에도 대처할 수 있는 완전한 개발자로 거듭나게 될 것이다.

-----

## 00\. Criteria API: 동적 쿼리를 생성하는 타입-세이프(Type-Safe) 방식

검색 기능이 있는 애플리케이션을 만든다고 상상해 보자. 사용자는 회원의 이름, 나이, 소속 팀 등 여러 조건 중 원하는 것만 선택하여 검색할 수 있다. 이름만으로 검색할 수도 있고, 이름과 나이를 함께 검색할 수도 있다. 이런 **동적 쿼리**를 JPQL 문자열로 처리하려면 어떻게 해야 할까?

```kotlin
// 동적 쿼리를 JPQL 문자열로 처리하는 고통스러운 방법
var jpql = "SELECT m FROM Member m"
val conditions = mutableListOf<String>()

if (name != null) {
    conditions.add("m.username = :name")
}
if (age != null) {
    conditions.add("m.age > :age")
}

if (conditions.isNotEmpty()) {
    jpql += " WHERE " + conditions.joinToString(" AND ")
}
//... 파라미터 바인딩 로직 추가...
```

조건이 몇 개 없을 때는 괜찮지만, 검색 조건이 10개가 넘어가면 문자열 조립 로직은 걷잡을 수 없이 복잡해지고, 사소한 오타 하나가 런타임에 치명적인 오류를 발생시킨다.

**Criteria API**는 바로 이 동적 쿼리 문제를 해결하기 위해 탄생했다. Criteria는 문자열이 아닌 **자바/코틀린 코드를 통해 프로그래밍 방식으로 쿼리를 생성**하는 JPA 표준 기능이다.

-----

### **Criteria의 타입-세이프(Type-Safe) 장점**

Criteria의 가장 큰 장점은 **타입-안전성**을 제공한다는 것이다. JPQL 문자열 쿼리는 `m.age = '문자열'` 과 같은 실수를 해도 컴파일 시점에는 알 수 없고, 오직 런타임에만 오류를 발견할 수 있다. 하지만 Criteria는 모든 쿼리 구성 요소를 자바/코틀린 객체와 메서드 호출로 표현하므로, 다음과 같은 오류를 컴파일러가 미리 잡아준다.

```kotlin
// cb.equal(m.get("age"), "문자열") // -> 컴파일 에러! age는 Int 타입이다.
```

이러한 타입-안전성 덕분에 훨씬 더 견고하고 안정적인 데이터 접근 코드를 작성할 수 있다.

-----

### **Criteria의 핵심 구성 요소**

Criteria 쿼리는 몇 가지 핵심 객체들의 조합으로 만들어진다.

  * `EntityManager`: 모든 것의 시작점. `getCriteriaBuilder()`를 통해 `CriteriaBuilder`를 얻는다.
  * `CriteriaBuilder`: 쿼리를 생성하는 팩토리(Factory) 객체. `createQuery()`, `select()`, `where()` 등 쿼리의 각 부분을 만드는 메서드를 제공한다.
  * `CriteriaQuery<T>`: 생성될 쿼리 자체를 나타내는 객체. 반환될 타입을 제네릭으로 지정한다.
  * `Root<T>`: 쿼리의 `FROM` 절에 해당하는 객체. 쿼리의 루트가 되는 엔티티를 나타내며, 필드에 접근하기 위한 시작점 역할을 한다.

**간단한 Criteria 쿼리 예시**

```kotlin
// JPQL: SELECT m FROM Member m WHERE m.username = 'Kim'

@Test
@Transactional
fun `Criteria 기본 쿼리 테스트`() {
    entityManager.persist(Member(username = "Kim", age = 30))

    // 1. CriteriaBuilder 획득
    val cb = entityManager.criteriaBuilder

    // 2. CriteriaQuery 생성 (반환 타입 지정)
    val cq = cb.createQuery(Member::class.java)

    // 3. FROM 절 (Root) 생성
    val m = cq.from(Member::class.java)

    // 4. SELECT 절과 WHERE 절 생성
    cq.select(m).where(cb.equal(m.get<String>("username"), "Kim"))

    // 5. 쿼리 실행
    val resultList = entityManager.createQuery(cq).resultList

    assertThat(resultList[0].username).isEqualTo("Kim")
}
```

`m.get<String>("username")` 부분에서 필드 이름을 문자열로 사용했는데, 이 방식은 오타에 취약하다. 이 문제를 해결하기 위해 '메타모델'이라는 것을 사용할 수 있다.

Criteria는 이처럼 타입-안전하고 프로그래밍 방식으로 동적 쿼리를 만들 수 있는 강력한 도구다. 하지만 예시 코드에서 볼 수 있듯이, 가장 간단한 쿼리조차도 JPQL에 비해 매우 장황하고 복잡하다는 명확한 단점을 가지고 있다. 다음 절에서는 이 장점과 단점을 더 자세히 파고들어 보겠다.