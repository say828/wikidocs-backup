## 03\. 테스트 코드를 활용한 보안 취약점(SQL Injection, XSS) 방어

빠르고 안정적인 시스템을 구축했다 하더라도, 단 한 번의 보안 사고는 회사의 신뢰와 비즈니스를 송두리째 무너뜨릴 수 있다. 보안은 더 이상 일부 보안 전문가의 전유물이 아니다. 코드를 작성하는 개발자 스스로가 잠재적인 위협을 이해하고 이를 방어해야 하는 \*\*'시큐어 코딩(Secure Coding)'\*\*은 이제 모든 개발자의 기본 소양이다.

TDD는 이러한 시큐어 코딩을 실천하는 매우 효과적인 방법이다. 우리는 "만약 해커가 이런 방식으로 공격한다면, 우리 시스템은 막아내야 한다"는 **공격 시나리오 자체를 실패하는 테스트 케이스로 먼저 작성**할 수 있다. 이는 개발자가 방어적인 관점에서 코드를 설계하도록 강제하며, 대표적인 웹 취약점인 SQL Injection이나 XSS(Cross-Site Scripting)를 효과적으로 예방하는 안전망을 제공한다.

### **SQL Injection 방어 테스트**

**SQL Injection**은 공격자가 악의적인 SQL 구문을 입력 값에 삽입하여 데이터베이스를 비정상적으로 조작하는 공격 기법이다. 이는 특히 사용자의 입력을 받아 동적으로 쿼리를 생성하는 부분에서 발생하기 쉽다.

**나쁜 예: SQL Injection에 취약한 코드 (문자열 접합 방식)**

```kotlin
// 절대 이렇게 쿼리를 만들지 마세요!
fun findProductsByName(name: String): List<Product> {
    val sql = "SELECT * FROM product WHERE name = '" + name + "'"
    // ... jdbcTemplate.query(sql, ...) ...
}
```

만약 공격자가 `name` 값으로 `' OR '1'='1` 과 같은 문자열을 입력하면, 최종 SQL은 `SELECT * FROM product WHERE name = '' OR '1'='1'` 이 되어 모든 상품 정보가 유출될 것이다.

**TDD를 통한 방어:**

우리는 바로 이 공격 시나리오를 테스트 케이스로 만든다.

**1. 실패하는 공격 시나리오 테스트 작성 (RED)**

```kotlin
@Test
fun `상품 이름에 SQL Injection 공격 구문이 포함되어도 안전해야 한다`() {
    // given
    val normalProduct = productRepository.save(Product(name = "정상 제품", ...))
    val secretProduct = productRepository.save(Product(name = "대외비 제품", ...))
    
    val injectionAttempt = "' OR '1'='1"

    // when
    // JPA Repository의 기본 구현은 Parameterized Query를 사용하므로 안전하다.
    // 만약 직접 구현한 메소드가 있다면 이 테스트는 중요하다.
    val foundProducts = productRepository.findByName(injectionAttempt)

    // then
    // 공격이 성공했다면 모든 제품(2개)이 반환될 것이다.
    // 공격이 실패했다면 아무것도 반환되지 않아야 한다.
    foundProducts.size shouldBe 0
}
```

만약 `findByName` 메소드가 문자열 접합 방식으로 구현되었다면, 이 테스트는 `foundProducts.size`가 2가 되어 **실패**한다.

**2. 안전한 코드로 테스트 통과 (GREEN)**

테스트를 통과시키는 방법은 간단하다. 절대로 사용자 입력을 SQL 문자열에 직접 접합하지 않고, 항상 \*\*Parameterized Query(준비된 문장, Prepared Statement)\*\*를 사용해야 한다. Spring Data JPA, MyBatis, jOOQ와 같은 현대적인 데이터 접근 프레임워크는 기본적으로 이 방식을 사용하므로 SQL Injection을 효과적으로 방어해준다.

```kotlin
// Spring Data JPA는 자동으로 안전한 코드를 생성한다.
interface ProductRepository : JpaRepository<Product, Long> {
    // 이 쿼리는 내부적으로 Parameterized Query로 변환된다.
    // SELECT * FROM product WHERE name = ?
    fun findByName(name: String): List<Product>
}
```

이제 다시 테스트를 실행하면, 프레임워크가 악의적인 입력값을 단순한 문자열로 처리하므로 아무런 상품도 찾지 못하고 `foundProducts.size`가 0이 되어 테스트는 **성공**한다.

### **XSS (Cross-Site Scripting) 방어 테스트**

XSS는 공격자가 악의적인 스크립트(주로 JavaScript)를 웹 페이지에 삽입하여 다른 사용자의 브라우저에서 실행되게 만드는 공격이다. 이를 통해 사용자의 세션 쿠키를 탈취하거나 개인정보를 빼낼 수 있다.

**TDD를 통한 방어:**

"사용자가 입력한 게시글 내용에 스크립트 태그가 포함되어 있다면, 시스템은 이를 위험하지 않은 문자열(HTML Entity)로 변환하여 저장해야 한다"는 요구사항을 테스트로 작성한다.

```kotlin
@Test
fun `게시글 내용에 포함된 스크립트는 HTML 이스케이프 처리되어야 한다`() {
    // given
    val maliciousContent = "<script>alert('hacked!')</script>"
    val postRequest = PostCreationRequest(title = "XSS Test", content = maliciousContent)
    
    // when
    val createdPost = postService.createPost(postRequest)
    
    // then
    val expectedContent = "&lt;script&gt;alert('hacked!')&lt;/script&gt;"
    
    // DB에 저장된 내용이 이스케이프 처리되었는지 검증
    createdPost.content shouldBe expectedContent
}
```

이 테스트를 통과시키기 위해, `postService` 내부에서는 사용자 입력을 저장하기 전에 **HTML Escape**를 수행하는 라이브러리(예: `org.apache.commons.text.StringEscapeUtils`)를 사용하거나, 프레임워크가 제공하는 보안 기능을 활용해야 한다.

-----

TDD를 보안 테스트에 활용하는 것은, 개발자가 잠재적인 공격자의 관점에서 자신의 코드를 바라보게 만드는 강력한 훈련이다. 우리는 더 이상 "설마 이런 입력이 들어오겠어?"라고 막연히 가정하는 대신, \*\*"만약 이런 공격이 들어온다면?"\*\*이라고 적극적으로 질문하고, 그에 대한 방어 코드를 테스트와 함께 구체적으로 증명하게 된다. 이는 시스템의 견고함을 한 차원 높은 수준으로 끌어올리는, 모든 TDD 전문가가 갖추어야 할 필수적인 역량이다.