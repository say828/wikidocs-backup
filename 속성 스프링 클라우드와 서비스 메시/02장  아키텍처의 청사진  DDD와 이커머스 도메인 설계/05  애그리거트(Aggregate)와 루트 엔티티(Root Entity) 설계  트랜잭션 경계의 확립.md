## 애그리거트(Aggregate)와 루트 엔티티(Root Entity) 설계: 트랜잭션 경계의 확립

지금까지 우리는 '전략적 설계(Strategic Design)'를 통해 서비스의 '경계'(BC)와 '관계'(Context Map), 그리고 '우선순위'(Core Domain)라는 거시적인 청사진을 그렸습니다.

이제 '전술적 설계(Tactical Design)'로 들어가, 이 청사진을 실제 '코드'로 구현하는 첫 번째 단계를 밟습니다. 그 핵심에 바로 \*\*'애그리거트(Aggregate)'\*\*가 있습니다.

-----

### 애그리거트란 무엇인가?

'주문(Order)' 도메인을 생각해 봅시다. '주문'이 생성된다는 것은, `Order` 객체(주문 정보)뿐만 아니라, 여러 개의 `OrderLine` 객체(주문 항목들), 그리고 `ShippingInfo` 객체(배송지 정보)가 \*\*'한 묶음'\*\*으로 생성된다는 의미입니다.

\*\*애그리거트(Aggregate)\*\*란, 이처럼 \*\*'데이터 변경의 일관성을 유지하기 위해 하나의 단위로 취급되어야 하는 객체들의 묶음(Cluster)'\*\*을 의미합니다.

### 애그리거트 루트 (Aggregate Root)

이 '객체 묶음'은 아무렇게나 접근해서는 안 됩니다. 묶음 내의 일관성(비즈니스 규칙)을 보장하기 위한 '단일 진입점(Gateway)'이 필요합니다. 이 진입점이 바로 \*\*애그리거트 루트(Aggregate Root)\*\*입니다.

  * **애그리거트 루트:** 애그리거트 묶음 전체를 대표하는 '루트 엔티티(Root Entity)'입니다.
  * **핵심 규칙 1:** 애그리거트 외부의 객체는 **오직 '루트'만을 참조**할 수 있습니다. (예: `Order`는 참조 OK, `OrderLine`은 직접 참조 NO)
  * **핵심 규칙 2:** 애그리거트 내부의 객체(예: `OrderLine`)를 변경하려면, 반드시 **'루트'를 통해서만** 가능해야 합니다.
  * **핵심 규칙 3:** 애그리거트의 모든 객체는 **하나의 트랜잭션**으로 저장/수정/삭제되어야 합니다.

-----

### "트랜잭션 경계의 확립"

애그리거트의 가장 중요한 존재 이유입니다.

> **"애그리거트 = 트랜잭션(Transaction)의 경계"**

애그리거트 루트(예: `Order`)를 `Repository.save(order)` 하는 순간, 이 애그리거트에 속한 모든 객체( `OrderLine`, `ShippingInfo` 등)는 **반드시 하나의 원자적(Atomic) 트랜잭션**으로 DB에 저장되어야 합니다.

만약 `Order`는 저장됐는데, 3개의 `OrderLine` 중 2개만 저장되고 1개가 누락된다면? 데이터가 깨지고 비즈니스가 망가집니다. 애그리거트는 이 '데이터 일관성'을 보장하는 최소한의 '단위'입니다.

### 이커머스 `Order` 애그리거트 설계

우리의 '핵심 도메인(Core Domain)'인 `주문(Order)`을 애그리거트로 설계해 봅시다.

**[ `Order` Aggregate ]**

  * **`Order` (Root Entity):** 애그리거트 루트. `id`, `orderer`(주문자), `status`(주문 상태), `shippingInfo`(배송지), `totalAmount`(총액)
  * **`OrderLine` (Entity):** 애그리거트 내부 엔티티. `productId`, `price`(주문 시점 가격), `quantity`(수량)
  * **`ShippingInfo` (Value Object):** `recipient`(수신자), `address`(주소), `phone`(연락처)
  * **`Orderer` (Value Object):** `memberId`

이 설계에는 매우 중요한 DDD 규칙이 숨어있습니다.

**규칙: 애그리거트는 다른 애그리거트를 'ID'로만 참조한다.**

`OrderLine`은 `Product` 객체 자체를 참조하지 않고, 오직 **`productId: Long`** 이라는 'ID' 값만 가지고 있습니다.

  * **왜?** 만약 `Order` 애그리거트가 `Product` 애그리거트 객체를 직접 참조( `product: Product` )한다면, 두 애그리거트가 강하게 결합됩니다.
  * `Order`를 조회할 때마다 불필요하게 `Product` 정보까지 로드해야 할 수도 있고(JPA N+1 문제), `Product`의 변경이 `Order` 트랜잭션에 영향을 줄 수 있습니다.
  * `Product`는 '상품' BC, `Order`는 '주문' BC 소속입니다. 서로 다른 BC의 애그리거트는 **ID로만 참조**하는 것이 경계를 명확히 하고 느슨한 결합(Loose Coupling)을 유지하는 핵심 비결입니다.

### 애그리거트 루트가 '비즈니스 규칙'을 강제하는 법

애그리거트 루트는 묶음의 '일관성 규칙(Invariants)'을 강제합니다.

  * **규칙(Invariant):** "배송(SHIPPED)이 시작된 `Order`에는 `OrderLine`을 추가할 수 없다."
  * **규칙(Invariant):** "`Order`의 `totalAmount`는 항상 모든 `OrderLine`의 (가격 \* 수량)의 합과 같아야 한다."

이 규칙을 어떻게 강제할까요? `OrderLine`을 직접 수정하는 것을 막고, `Order` 루트를 통하도록 강제합니다.

**[나쁜 코드 👎]**

```kotlin
// '루트'를 통하지 않고 'OrderLine'을 직접 추가하려는 시도
// (orderLineRepository가 있다고 가정. 애초에 만들면 안 됨)
val line = OrderLine(productId = 1, price = 1000, quantity = 1)
orderLineRepository.save(line) // ⛔️ 금지! 'Order'의 totalAmount와 불일치 발생
```

**[좋은 코드 (애그리거트 루트 사용) 👍]**

```kotlin
// 1. 애그리거트 루트를 조회
val order = orderRepository.findById(orderId)

// 2. '루트'의 메서드를 통해 내부 객체(OrderLine)를 조작
order.addOrderLine(productId = 1, price = 1000, quantity = 1)

// 3. '루트'를 저장 (이때 Order, OrderLine이 한 트랜잭션으로 저장됨)
orderRepository.save(order)

// --- Order 클래스 (애그리거트 루트) ---
@Entity
class Order(
    // ...
    @OneToMany(cascade = [CascadeType.ALL], orphanRemoval = true)
    val orderLines: MutableList<OrderLine> = mutableListOf(),
    
    var totalAmount: Long = 0,
    var status: OrderStatus
) {
    /**
     * 애그리거트 루트가 '일관성 규칙(Invariants)'을 강제한다.
     */
    fun addOrderLine(productId: Long, price: Long, quantity: Int) {
        // [규칙 1 검사] 배송 중이면 추가 불가
        if (this.status == OrderStatus.SHIPPED || this.status == OrderStatus.CANCELLED) {
            throw IllegalStateException("이미 배송되었거나 취소된 주문입니다.")
        }
        
        // 내부 객체 추가
        val newOrderLine = OrderLine(productId, price, quantity)
        this.orderLines.add(newOrderLine)
        
        // [규칙 2 적용] 총액 자동 계산
        recalculateTotalAmount()
    }

    private fun recalculateTotalAmount() {
        // 모든 'OrderLine'의 합계를 계산하여 'totalAmount'를 갱신
        this.totalAmount = orderLines.sumOf { it.price * it.quantity }
    }
}
```

-----

이것으로 02장을 마칩니다. 우리는 '이커머스'라는 복잡한 도메인을 분석하여 '유비쿼터스 랭귀지'를 정의했고, 서비스 경계인 '바운디드 컨텍스트' 5개를 식별했습니다. 서비스 간 관계 '컨텍스트 맵'을 그렸고, 리소스 집중을 위한 '핵심 도메인'을 선정했습니다. 마지막으로 데이터 일관성의 단위인 '애그리거트'까지 설계했습니다.

이 \*\*완벽한 '청사진'\*\*을 가지고, 이제 03장부터 실제 '코드'로 첫 번째 서비스를 구축해 나갈 시간입니다.