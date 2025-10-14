## 불변성(val)과 데이터 클래스(data class)를 활용한 깨끗한 DTO/도메인 모델 설계

스프링 부트 애플리케이션에서 가장 많이 작성하는 클래스는 아마도 '데이터'를 담는 객체일 것입니다. 컨트롤러(Controller) 계층에서 요청을 받는 `DTO(Data Transfer Object)`, 그리고 DB 테이블과 매핑되는 `엔티티(Entity)`나 핵심 비즈니스 로직을 담는 `도메인 모델`이 바로 그것입니다.

코틀린은 이 '데이터' 객체를 다루는 데 있어 자바와 비교할 수 없는 강력함과 간결함을 제공합니다. 핵심은 \*\*`val` (불변성)\*\*과 \*\*`data class` (데이터 클래스)\*\*입니다.

-----

### 불변성(val): 예측 가능한 시스템의 첫걸음

전통적인 자바 빈(JavaBean) 스타일의 코드는 `setter` 메서드를 통해 객체의 상태를 자유롭게 변경(Mutable)할 수 있도록 설계되었습니다.

```java
// Java - 변경 가능한(Mutable) DTO
public class ProductRequest {
    private String name;
    private Long price;
    
    // public getters and setters...
    public void setName(String name) {
        this.name = name;
    }
    //...
}

// 어딘가의 서비스 로직
public void processProduct(ProductRequest request) {
    // 요청(request) 객체의 상태가 원본과 달라짐
    request.setName(request.getName().trim()); 
    
    if (request.getPrice() < 0) {
        // 심지어 가격도 바꿈
        request.setPrice(0L);
    }
    // ...
    // 이 메서드를 호출한 상위 메서드는 request 객체가 변경되었음을 알기 어렵다.
}
```

이러한 '변경 가능성'은 복잡한 MSA 환경에서 심각한 버그의 원인이 됩니다. 데이터가 여러 서비스나 스레드로 전달되는 과정에서 원본 객체의 상태가 예기치 않게 변경된다면(Side Effect), 시스템의 동작을 예측하고 디버깅하는 것은 지옥에 가까워집니다.

코틀린은 \*\*`val` (value)\*\*을 사용해 '읽기 전용' (정확히는 재할당 불가) 프로퍼티를 선언하도록 강력히 권장합니다.

```kotlin
// Kotlin - 불변(Immutable) DTO
class ProductRequest(
    val name: String,
    val price: Long
)

fun processProduct(request: ProductRequest) {
    // request.name = "New Name" // 컴파일 에러! 'val'은 재할당될 수 없습니다.

    // 만약 값을 변경해야 한다면, '새로운' 객체를 생성해야 합니다.
    val processedRequest = request.copy(
        name = request.name.trim(),
        price = if (request.price < 0) 0L else request.price
    )
    // ...
    // 원본 'request' 객체는 절대 변경되지 않음이 보장됩니다.
}
```

이처럼 '불변 객체'를 사용하면, 데이터가 시스템의 어느 곳을 흘러 다니든 그 상태가 '원본' 그대로임을 보장할 수 있어, 시스템 전체의 안정성과 예측 가능성이 극적으로 향상됩니다.

-----

### 데이터 클래스(data class): 상용구 코드의 완벽한 제거

앞서 `val`을 사용한 DTO 예제는 자바보다 간결하지만, 두 객체가 같은지 비교하는 `equals()`, 객체의 내용을 로그로 찍기 위한 `toString()`, 컬렉션에서 키로 사용하기 위한 `hashCode()` 등의 메서드가 빠져있습니다.

자바에서는 이를 위해 Lombok의 `@Data`나 `@Value`를 사용하거나, IDE의 도움을 받아 수십 줄의 코드를 생성해야 했습니다.

코틀린은 이 모든 것을 `data`라는 키워드 하나로 해결합니다.

```kotlin
// Kotlin - 단 한 줄의 완성형 DTO
data class ProductRequest(
    val name: String,
    val price: Long
)
```

컴파일러는 `data class` 키워드를 보고 다음과 같은 메서드들을 **자동으로** 생성해 줍니다.

1.  **`equals()`**: 주 생성자에 나열된 프로퍼티(`name`, `price`)의 값이 같으면 `true`를 반환합니다.
2.  **`hashCode()`**: `equals()`와 일관성을 가지는 해시코드를 생성합니다.
3.  **`toString()`**: `ProductRequest(name=..., price=...)`와 같이 읽기 편한 문자열을 반환합니다.
4.  **`copy()`**: 객체의 일부 프로퍼티만 변경하여 '새로운' 불변 객체를 생성할 때 사용하는 강력한 메서드입니다. (위 `processProduct` 예제에서 사용됨)
5.  **`componentN()`**: 구조 분해 할당(Destructuring Declarations)을 위한 메서드입니다.

**이커머스 도메인 모델 적용**

이러한 특성은 '회원', '상품', '주문' 같은 우리의 핵심 도메인 모델을 설계할 때도 빛을 발합니다. (JPA `Entity` 설계 시에는 몇 가지 주의사항이 있으며, 이는 01장 02절에서 자세히 다룹니다.)

```kotlin
// 주문(Order) 도메인의 핵심 정보를 나타내는 모델
data class OrderModel(
    val orderId: Long,
    val memberId: Long,
    val orderLines: List<OrderLine>, // 불변 리스트(List) 사용
    val status: OrderStatus
)

data class OrderLine(
    val productId: Long,
    val price: Long,
    val quantity: Int
)

enum class OrderStatus {
    PENDING, PAID, SHIPPED, CANCELLED
}
```

`data class`와 `val`의 조합은 JPA 엔티티를 제외한 거의 모든 DTO와 도메인 객체를 설계하는 표준 방식이 될 것입니다. 이는 개발자의 생산성을 극대화하는 동시에, '불변성'을 통해 '안정성'이라는 두 마리 토끼를 잡는 코틀린의 핵심 철학입니다.