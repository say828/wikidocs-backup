## 02\. Spock 프레임워크: Groovy를 활용한 또 다른 테스트 선택지

이 책은 코틀린 생태계에 가장 잘 어울리는 Kotest를 메인 테스트 프레임워크로 선택했다. 하지만 JVM 세계에는 TDD와 BDD(행위 주도 개발)의 철학을 매우 우아하게 구현한 또 다른 강력한 강자가 존재한다. 바로 **Spock 프레임워크**다. Spock은 코틀린이 아닌 **Groovy** 언어를 사용하여 테스트를 작성하며, 간결한 문법과 강력한 기능으로 오랫동안 많은 개발자들의 사랑을 받아왔다.

TDD 전문가로서 다양한 도구를 이해하고 상황에 맞는 최적의 도구를 선택할 수 있는 시야를 갖추는 것은 매우 중요하다. Spock은 Kotest와는 또 다른 매력을 가진, 반드시 알아둘 가치가 있는 선택지다.

### **Spock의 핵심 특징과 구조**

Spock 테스트는 `spock.lang.Specification` 클래스를 상속받아 작성하며, 그 구조는 명확하게 구분된 \*\*블록(Block)\*\*들로 이루어진다.

  * `given:` (또는 `setup:`): 테스트에 필요한 객체를 설정하고 초기화하는 '준비' 블록.
  * `when:`: 테스트하려는 실제 '행위'가 일어나는 블록.
  * `then:`: 행위의 '결과'를 검증하는 블록. Spock에서는 `assert` 키워드 없이, boolean을 반환하는 모든 표현식이 자동으로 단언문이 된다.
  * `where:`: 데이터 주도 테스트를 위한 '입력 데이터 테이블'을 정의하는 블록.

<!-- end list -->

```groovy
import spock.lang.Specification

class CalculatorSpec extends Specification {
    
    def "두 숫자를 더하면 그 합을 반환해야 한다"() {
        given: "계산기 인스턴스가 주어졌을 때"
        def calculator = new Calculator()

        when: "3과 5를 더하면"
        def result = calculator.add(3, 5)

        then: "결과는 8이어야 한다"
        result == 8
    }
}
```

### **Spock의 강력한 기능들**

#### **1. 마법과 같은 데이터 주도 테스트 (`where` 블록)**

Spock이 찬사를 받는 가장 큰 이유 중 하나는 바로 `where` 블록을 이용한 데이터 주도 테스트다. 여러 입력값과 기대값을 마치 표처럼 직관적으로 표현할 수 있다.

```groovy
def "다양한 숫자를 더하는 경우들을 테스트한다"() {
    expect: "a와 b를 더하면 c가 되어야 한다"
    calculator.add(a, b) == c

    where: "다음과 같은 입력값과 기대값이 주어졌을 때"
    a | b  || c
    3 | 5  || 8
    0 | 10 || 10
    -5| 5  || 0
    -1| -3 || -4
}
```

이 간결하고 가독성 높은 문법은 여러 경계값과 예외 케이스를 테스트해야 할 때 엄청난 생산성을 제공한다.

#### **2. 내장된 Mocking과 Stubbing 프레임워크**

Spock은 Mockito나 MockK와 같은 별도의 Mocking 라이브러리가 필요 없다. 프레임워크 자체에 강력하고 직관적인 Mocking/Stubbing 기능이 내장되어 있다.

```groovy
def "회원 가입 시 이메일 클라이언트를 호출해야 한다"() {
    given: "이메일 클라이언트 Mock 객체가 주어진다"
    def emailClient = Mock(EmailClient)
    def userService = new UserService(emailClient)

    when: "사용자가 가입하면"
    userService.register("test@spock.com")

    then: "emailClient의 send 메소드가 정확히 한 번 호출되어야 한다"
    1 * emailClient.send("test@spock.com", "환영합니다!")
}

def "상품 조회 시 Repository Stub이 고정된 값을 반환한다"() {
    given: "상품 리포지토리 Stub 객체가 주어진다"
    def productRepo = Stub(ProductRepository)
    // findById 메소드가 어떤 id로 호출되든 고정된 Product 객체를 반환
    productRepo.findById(_) >> new Product(name: "Spock Book", price: 30000)
    
    def productService = new ProductService(productRepo)

    when: "상품 정보를 조회하면"
    def product = productService.getProductInfo(123)

    then: "Stub이 반환한 상품 정보가 그대로 나온다"
    product.name == "Spock Book"
}
```

`Mock()`으로 행위 검증용 객체를, `Stub()`으로 상태 제공용 객체를 만들고, `>>` (right shift) 연산자로 반환 값을, `*` 연산자로 호출 횟수를 검증하는 문법은 매우 직관적이다.

### **Spock vs. Kotest**

| 특징 | Spock | Kotest |
| :--- | :--- | :--- |
| **언어** | Groovy | Kotlin |
| **데이터 주도 테스트** | 매우 간결하고 강력한 `where` 블록 | `withData`를 사용, 상대적으로 장황 |
| **Mocking** | 내장 (자체 프레임워크) | 외부 라이브러리 필요 (MockK) |
| **코루틴 지원** | 제한적 | 완벽 지원 (Native) |
| **생태계** | Java/Groovy 프로젝트에 유리 | Kotlin 프로젝트에 유리 |

**전문가의 선택:**
당신의 팀이 코틀린 우선(Kotlin-first) 환경에 있고, 코루틴과 같은 코틀린의 최신 기능을 적극적으로 사용한다면 **Kotest**가 더 자연스러운 선택이다. 반면, Java와 Groovy 기반의 프로젝트가 많거나, 무엇보다 간결한 데이터 주도 테스트를 선호한다면 **Spock**은 매우 매력적이고 강력한 대안이다.

궁극적으로 어떤 도구를 선택하든, 테스트를 통해 행위를 명세하고, 설계를 개선하며, 자신감을 얻는다는 TDD의 핵심 철학은 동일하다. Spock은 그 철학을 구현하는 또 하나의 훌륭한 길을 제시하는 도구로 이해하는 것이 좋다.