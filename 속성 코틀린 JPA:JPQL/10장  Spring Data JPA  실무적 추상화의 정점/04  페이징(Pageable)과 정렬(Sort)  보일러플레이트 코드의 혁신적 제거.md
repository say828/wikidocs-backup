## 04\. 페이징(Pageable)과 정렬(Sort): 보일러플레이트 코드의 혁신적 제거

6장에서 JPA의 페이징 API(`setFirstResult`, `setMaxResults`)를 배웠다. 분명 동작은 하지만, 페이지 번호로부터 시작 위치를 계산하는 `(pageNumber - 1) * pageSize` 같은 보일러플레이트 코드를 매번 작성해야 하는 번거로움이 있었다. 정렬(Sorting)은 더 심해서, JPQL 문자열에 `ORDER BY` 절을 직접 추가해야 했다.

스프링 데이터 JPA는 이 페이징과 정렬 로직을 \*\*`Pageable`\*\*과 \*\*`Sort`\*\*라는 두 개의 매우 잘 만들어진 인터페이스로 완벽하게 추상화하여, 이 모든 번거로운 작업을 코드에서 완전히 몰아냈다. 🌊

-----

### **`Sort`: 객체지향적인 정렬**

`Sort` 객체를 사용하면, `ORDER BY` 절을 문자열로 다루는 대신 객체지향적으로 정렬 조건을 조합할 수 있다.

```kotlin
// username을 기준으로 내림차순 정렬
val sort = Sort.by("username").descending()
```

이 `Sort` 객체를 리포지토리 메서드의 파라미터로 추가하기만 하면, 스프링 데이터 JPA가 알아서 쿼리에 `ORDER BY` 절을 추가해준다.

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {
    // Sort 객체를 파라미터로 받는다.
    fun findByAgeGreaterThan(age: Int, sort: Sort): List<Member>
}

// 서비스 코드에서 호출
val members = memberRepository.findByAgeGreaterThan(20, Sort.by("username").descending())
// 실행되는 JPQL: SELECT m FROM Member m WHERE m.age > ?1 ORDER BY m.username DESC
```

-----

### **`Pageable`: 페이징과 정렬의 완성**

**`Pageable`** 인터페이스는 페이징(페이지 번호, 페이지 크기)과 정렬(`Sort`) 정보를 모두 담고 있는 강력한 객체다. 우리는 `PageRequest.of(...)` 정적 팩토리 메서드를 통해 `Pageable`의 구현체를 쉽게 만들 수 있다.

```kotlin
// 0번째 페이지, 페이지당 10개, username 내림차순 정렬
val pageable = PageRequest.of(0, 10, Sort.by("username").descending())
```

이 `pageable` 객체를 리포지토리 메서드의 파라미터로 넘기면, 마법이 일어난다.

```kotlin
interface MemberRepository : JpaRepository<Member, Long> {
    // Pageable 객체를 파라미터로 받고, Page<Member>를 반환한다.
    fun findByAgeGreaterThan(age: Int, pageable: Pageable): Page<Member>
}
```

### **`Page<T>`: 페이징 결과를 위한 완벽한 선물 상자**

`Pageable`을 파라미터로 사용하는 메서드는 단순히 `List<T>`를 반환하지 않고, \*\*`Page<T>`\*\*라는 특별한 객체를 반환한다. 이 `Page<T>` 객체 안에는 페이징 UI를 구현하는 데 필요한 거의 모든 정보가 담겨있다. 🎁

  * `getContent()`: 현재 페이지의 데이터 리스트 (`List<T>`)
  * `getTotalElements()`: 조건에 맞는 **전체 데이터 개수** (페이징과 무관)
  * `getTotalPages()`: **전체 페이지 수**
  * `getNumber()`: 현재 페이지 번호 (0부터 시작)
  * `getSize()`: 페이지 크기
  * `isFirst()`, `isLast()`: 첫 페이지 또는 마지막 페이지 여부
  * `hasNext()`, `hasPrevious()`: 다음 또는 이전 페이지 존재 여부

`getTotalElements()`를 가져오기 위해 별도의 `count` 쿼리가 자동으로 한 번 더 실행된다. 이 `Page` 객체 하나만 있으면, 개발자는 페이지네이션 UI를 만드는 데 필요한 그 어떤 추가 작업도 할 필요가 없다.

**코드 예시**

```kotlin
@Test
@Transactional
fun `페이징과 정렬 테스트`() {
    // ... 100명의 멤버 저장 ...

    // 2페이지, 5개씩, 나이 내림차순
    val pageable = PageRequest.of(1, 5, Sort.by("age").descending())
    
    val page: Page<Member> = memberRepository.findByAgeGreaterThan(0, pageable)

    val content: List<Member> = page.content // 조회된 데이터

    assertThat(content).hasSize(5)
    assertThat(page.totalElements).isEqualTo(100)
    assertThat(page.number).isEqualTo(1)
    assertThat(page.totalPages).isEqualTo(20)
    assertThat(page.isFirst).isFalse()
    assertThat(page.hasNext()).isTrue()
}
```

스프링 데이터 JPA의 `Pageable`과 `Page`는 페이징 처리와 관련된 모든 보일러플레이트 코드를 혁신적으로 제거하고, 개발자가 가장 중요한 비즈니스 로직에만 집중할 수 있도록 해주는 최고의 추상화 기능이다.

지금까지 우리는 정적인 쿼리를 다루는 다양한 방법을 배웠다. 하지만 실무의 가장 큰 난관은 바로 '동적 쿼리'다. 다음 절에서는 이 동적 쿼리를 해결하기 위한 첫 번째 방법인 '명세(Specification)'에 대해 알아본다.