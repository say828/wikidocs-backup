## 02\. 다형성 쿼리: `TYPE` 키워드를 이용한 상속 엔티티 조회

JPQL은 객체지향 언어답게 상속 관계를 완벽하게 이해하고 지원한다. 5장에서 배운 상속 매핑 전략(JOINED, SINGLE\_TABLE)을 사용하고 있을 때, JPQL은 매우 강력한 **다형성(Polymorphism)** 쿼리 기능을 제공한다.

-----

### **부모 타입으로 자식까지 조회하기**

만약 우리가 부모 클래스인 `Item`으로 JPQL 쿼리를 실행하면 어떤 결과가 나올까?

```jpql
SELECT i FROM Item i
```

JPA는 이 쿼리를 보고, `Item`뿐만 아니라 `Item`을 상속받은 모든 자식 엔티티(`Book`, `Album`, `Movie` 등)를 **함께 조회**하여 반환한다. 이는 객체지향의 다형성 원칙이 쿼리에도 그대로 적용된 것이다. `Item`을 조회했지만, 실제 반환된 리스트에는 `Book` 객체와 `Album` 객체가 섞여있을 수 있다.

`JOINED` 전략의 경우 `ITEM` 테이블을 기준으로 모든 자식 테이블을 `OUTER JOIN`하는 SQL이 실행되고, `SINGLE_TABLE` 전략의 경우 `ITEM` 테이블 전체를 조회하는 SQL이 실행된다.

-----

### **`TYPE` 키워드로 특정 자식 타입만 필터링**

때로는 `Item` 리스트 중에서 `Book`이나 `Movie`처럼 특정 타입의 엔티티만 골라서 조회하고 싶을 때가 있다. 이때 사용하는 것이 바로 **`TYPE`** 키워드다. `TYPE`은 `WHERE` 절에서 엔티티의 타입을 조건으로 사용할 수 있게 해준다.

**JPQL 문법**

```jpql
-- Item 중에서 Book 타입인 것만 조회
SELECT i FROM Item i WHERE TYPE(i) = Book
```

**코드 예시**

```kotlin
@Test
@Transactional
fun `TYPE 키워드를 사용한 다형성 쿼리 테스트`() {
    val book = Book(name = "JPA Book", author = "Kim", isbn = "1234")
    val album = Album(name = "Classic Album", artist = "Mozart")
    entityManager.persist(book)
    entityManager.persist(album)

    // Item 엔티티 중에서 타입이 Book인 것만 조회한다.
    val jpql = "SELECT i FROM Item i WHERE TYPE(i) = Book"
    val resultList = entityManager.createQuery(jpql, Item::class.java).resultList

    assertThat(resultList).hasSize(1)
    assertThat(resultList[0]).isInstanceOf(Book::class.java)
    val foundBook = resultList[0] as Book
    assertThat(foundBook.author).isEqualTo("Kim")
}
```

`TYPE(i) = Book` 조건을 통해, `Item`의 수많은 자식들 중 정확히 `Book` 타입의 엔티티만 필터링하여 가져올 수 있다.

-----

### **`TREAT` 키워드로 자식 타입 속성 사용하기 (JPA 2.1+)**

다형성 쿼리를 사용하다 보면, 부모 타입으로 조회했지만 `WHERE` 절에서는 특정 자식 타입의 필드를 조건으로 사용하고 싶을 때가 있다. 예를 들어, `Item` 중에서 `Book`이면서 저자(`author`)가 'Kim'인 엔티티를 찾고 싶다고 가정해 보자. `SELECT i FROM Item i WHERE i.author = 'Kim'` 과 같은 쿼리는 `Item` 엔티티에 `author` 필드가 없으므로 문법 오류다.

이때 사용하는 것이 바로 **`TREAT`** 키워드다. `TREAT`는 JPQL에서 \*\*자바/코틀린의 타입 캐스팅(형변환)\*\*과 같은 역할을 한다.

**JPQL 문법**

```jpql
-- Item을 Book으로 "취급(TREAT)"하여 Book의 고유 필드인 author를 사용한다.
SELECT i FROM Item i WHERE TREAT(i AS Book).author = 'Kim'
```

`TREAT(i AS Book)` 구문을 통해 JPA는 `i`를 `Book` 타입으로 간주하고, 그 뒤에 점(`.`)을 찍어 `Book`의 고유 필드인 `author`에 접근할 수 있게 해준다. 이 쿼리는 SQL로 변환될 때 `DTYPE`을 검사하고 `BOOK` 테이블과 조인하여 `author`를 비교하는 조건절을 자동으로 생성해준다.

`TYPE`과 `TREAT`는 상속 구조를 가진 엔티티 모델을 매우 유연하고 객체지향적으로 다룰 수 있게 해주는 JPQL만의 강력한 기능이다. 이제 다음 절에서는 JPQL이 기본으로 제공하는 다양한 내장 함수들에 대해 알아보자.