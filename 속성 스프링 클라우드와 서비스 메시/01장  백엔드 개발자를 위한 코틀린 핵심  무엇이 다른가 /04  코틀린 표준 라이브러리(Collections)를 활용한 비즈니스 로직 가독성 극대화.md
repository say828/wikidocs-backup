## 코틀린 표준 라이브러리(Collections)를 활용한 비즈니스 로직 가독성 극대화

이커머스 시스템의 비즈니스 로직을 구현한다는 것은 결국 '데이터 묶음(Collection)'을 다루는 일의 연속입니다.

  * '주문'에 포함된 '상품 목록'에서 특정 조건의 상품만 필터링하기
  * '상품 목록'의 가격과 수량을 곱해 '총 주문 금액' 계산하기
  * '상품 ID 리스트'를 '상품 정보 맵(Map)'으로 변환하여 쉽게 조회하기
  * '상품 리스트'를 '카테고리'별로 그룹화하기

자바 8 이전에는 `for` 루프와 `if` 문이 뒤섞인 '명령형(Imperative)' 코드로 이 모든 것을 처리해야 했습니다.

```java
// Java (Before Stream)
// 1. 주문 총액 계산
long totalAmount = 0;
for (OrderLine line : order.getOrderLines()) {
    totalAmount += line.getPrice() * line.getQuantity();
}

// 2. 특정 상품 ID 목록 추출
List<Long> productIds = new ArrayList<>();
for (OrderLine line : order.getOrderLines()) {
    if (line.getStatus() == OrderLineStatus.CONFIRMED) {
        productIds.add(line.getProductId());
    }
}
```

자바 8의 스트림(Stream) API가 도입되면서 `stream().filter().map().collect(...)`와 같이 '선언형(Declarative)' 방식이 가능해졌지만, `.stream()`을 매번 호출해야 하고 `Collectors` 문법이 다소 장황한 단점이 있습니다.

코틀린은 이 '컬렉션 처리'를 언어의 표준 라이브러리 레벨에서 훨씬 더 직관적이고 강력하게 지원합니다. 별도의 `.stream()` 호출 없이, 마치 원래 컬렉션의 기능인 것처럼 자연스럽게 연산자들을 체이닝(chaining)할 수 있습니다.

-----

### 이커머스 비즈니스 로직 예시

`Order`와 `OrderLine`이 다음과 같다고 가정해 봅시다.

```kotlin
data class Order(
    val id: Long,
    val orderLines: List<OrderLine>
)

data class OrderLine(
    val productId: Long,
    val price: Long,       // 개당 가격
    val quantity: Int,     // 수량
    val status: OrderLineStatus
)

enum class OrderLineStatus { PENDING, CONFIRMED, SHIPPED }
```

#### 1\. 총 주문 금액 계산: `sumOf`

가장 고전적인 `for` 루프와 `total += ...` 로직은 코틀린의 `sumOf` (또는 `sumOfLong`, `sumOfDouble`)로 완벽하게 대체됩니다.

```kotlin
// "주문 라인의 (가격 * 수량)의 합계를 구하라"
val totalAmount = order.orderLines.sumOf { it.price * it.quantity }
```

#### 2\. 특정 조건의 상품 ID 목록 추출: `filter` + `map`

'확정(CONFIRMED)'된 주문 라인의 '상품 ID' 목록만 필요할 수 있습니다.

```kotlin
// "주문 라인들 중에서(filter), 상태가 CONFIRMED인 것들만 골라,
//  그것들의 productId로 변환한 리스트(map)를 만들어라"
val confirmedProductIds: List<Long> = order.orderLines
    .filter { it.status == OrderLineStatus.CONFIRMED }
    .map { it.productId }
```

#### 3\. 데이터 변환 및 빠른 조회: `associateBy`

`상품` 서비스에서 `List<Product>`를 반환받았다고 가정해 봅시다. `주문` 서비스에서는 이 리스트를 `productId`로 빠르게 조회해야 합니다. `for` 루프를 돌며 `Map`을 생성할 수도 있지만, `associateBy`를 쓰면 단 한 줄로 끝납니다.

```kotlin
val products: List<Product> = productRepository.findAllById(confirmedProductIds)

// "상품 리스트를(associateBy), 각 상품의 'id'를 키로 하는 Map으로 변환하라"
// 결과: Map<Long, Product>
val productMap: Map<Long, Product> = products.associateBy { it.id }

// 이제 O(1) 시간 복잡도로 상품 정보를 조회할 수 있습니다.
val firstProductName = productMap[confirmedProductIds.first()]?.name
```

#### 4\. 데이터 그룹화: `groupBy`

'상품' 리스트를 '카테고리'별로 묶어서 화면에 보여줘야 할 때 유용합니다.

```kotlin
// "모든 상품 리스트를(groupBy), 각 상품의 'category'를 키로 하여 그룹화하라"
// 결과: Map<Category, List<Product>>
val productsByCategory: Map<Category, List<Product>> = allProducts
    .groupBy { it.category }
```

#### 5\. 안전한 단일 객체 조회: `firstOrNull` (or `find`)

특정 조건을 만족하는 첫 번째 객체를 찾을 때, `NPE`를 방지하는 `firstOrNull`은 필수입니다.

```kotlin
// "주문 라인 중에서(firstOrNull), 배송(SHIPPED) 상태인 첫 번째 항목을 찾아라.
//  만약 없으면, 예외를 터뜨리지 말고 null을 반환하라"
val firstShippedItem: OrderLine? = order.orderLines
    .firstOrNull { it.status == OrderLineStatus.SHIPPED }
```

-----

이처럼 코틀린 표준 라이브러리를 활용하면, 복잡한 '명령형' `for` 루프와 `if` 조건문이 **'비즈니스 요구사항을 설명하는' 선언형 코드로** 바뀝니다.

이는 코드의 가독성을 극대화하고, 버그를 줄이며, 유지보수를 매우 쉽게 만듭니다. 스프링 서비스 계층의 복잡한 비즈니스 로직을 이보다 더 깔끔하게 표현할 수 있는 방법은 없습니다.