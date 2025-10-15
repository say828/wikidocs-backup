## 04\. JPA와 DDD(Domain-Driven Design): Aggregates와 Repositories 설계 전략

지금까지 우리는 JPA를 '기술'의 관점에서 바라봤다. 하지만 JPA는 단순히 데이터를 저장하고 조회하는 도구를 넘어, 우리의 애플리케이션 아키텍처를 더 나은 방향으로 이끌 수 있는 '설계 사상'을 담고 있다. 특히 \*\*도메인 주도 설계(Domain-Driven Design, DDD)\*\*의 관점에서 JPA의 리포지토리를 바라보면, 우리는 코드의 응집도와 비즈니스 표현력을 극적으로 끌어올릴 수 있다.

-----

### **DDD의 핵심 패턴: 애그리거트(Aggregate)**

DDD의 핵심 개념 중 하나는 \*\*애그리거트(Aggregate)\*\*다. 애그리거트란 **'일관성의 경계'가 되는 엔티티들의 묶음**이다. 쉽게 말해, "A가 변경될 때 B와 C도 항상 함께 변경되어 하나의 비즈니스 규칙을 만족해야 한다"면, A, B, C는 하나의 애그리거트로 묶일 수 있다.

  * **애그리거트 루트(Aggregate Root)**: 애그리거트의 대표 엔티티. 이 애그리거트의 모든 데이터 변경은 반드시 **루트 엔티티를 통해서만** 이루어져야 한다. 외부에서는 애그리거트 내부의 다른 엔티티에 직접 접근할 수 없다.
  * **불변식(Invariants)**: 애그리거트 전체가 항상 만족해야 하는 비즈니스 규칙.

예를 들어, '주문(Order)' 애그리거트를 생각해 보자.

  * **루트**: `Order` 엔티티.
  * **내부 엔티티**: `OrderItem`, `ShippingInfo`
  * **불변식**: "주문 총액은 모든 주문 항목(OrderItem)의 합계와 같아야 한다."

`OrderItem`을 추가하거나 삭제하는 모든 작업은 반드시 `Order` 루트를 통해서만(`order.addOrderItem(...)`) 이루어져야 한다. 외부에서 `OrderItem`에 직접 접근하여 가격을 바꾸는 것은 금지된다. 이를 통해 애그리거트는 자신의 데이터 일관성을 스스로 책임질 수 있게 된다.

-----

### **JPA로 애그리거트 구현하기**

이 애그리거트 패턴은 JPA의 기능과 놀라울 정도로 잘 맞아떨어진다.

1.  **리포지토리는 애그리거트 루트에 대해서만 만든다.**
    DDD에서 리포지토리의 역할은 애그리거트 단위로 데이터를 조회하고 저장하는 것이다. 따라서 `OrderRepository`는 존재해야 하지만, `OrderItemRepository`나 `ShippingInfoRepository`는 **만들지 않는 것이 원칙**이다. `OrderItem`이 필요하다면, `OrderRepository`를 통해 `Order`를 조회한 뒤, `order.getOrderItems()`를 통해 접근해야 한다.

2.  **생명주기는 Cascade로 관리한다.**
    애그리거트는 하나의 단위로 생성되고 삭제된다. `Order`가 저장될 때 `OrderItem`도 함께 저장되어야 하고, `Order`가 삭제되면 `OrderItem`도 함께 삭제되어야 한다. 바로 이럴 때 12장에서 배운 \*\*`cascade = CascadeType.ALL`\*\*과 \*\*`orphanRemoval = true`\*\*가 빛을 발한다. 애그리거트 루트에서 내부 엔티티로의 연관관계에 이 옵션들을 적용하면, 애그리거트의 생명주기를 완벽하게 일치시킬 수 있다.

**Order.kt (애그리거트 루트)**

```kotlin
@Entity
class Order {
    // ...
    @OneToMany(mappedBy = "order", cascade = [CascadeType.ALL], orphanRemoval = true)
    val orderItems: MutableList<OrderItem> = mutableListOf()
    
    fun addOrderItem(item: Item, count: Int) {
        val orderItem = OrderItem(order = this, item = item, count = count)
        this.orderItems.add(orderItem)
        // 주문 총액 계산 등 불변식 유지 로직
        calculateTotalPrice()
    }
}
```

-----

> **결론: 리포지토리는 애그리거트의 관문이다.**
>
> JPA 리포지토리를 설계할 때, 단순히 모든 엔티티에 대해 리포지토리를 만드는 기계적인 접근법을 버려야 한다. 대신, 비즈니스 도메인을 분석하여 **애그리거트와 그 루트를 먼저 식별**하라. 그리고 **오직 애그리거트 루트에 대해서만 리포지토리를 생성**하라.
>
> 이 원칙을 지키는 것만으로도, 여러분의 코드는 훨씬 더 응집력 있고, 비즈니스 규칙을 잘 표현하며, 데이터의 일관성을 강력하게 보장하는 방향으로 진화할 것이다.

이제 아키텍처의 거시적인 관점에서 다시 JPA의 미시적인 세계로 돌아와, 영속성 컨텍스트가 실제로 살아있는 '범위'에 대한 마지막 주제를 다음 절에서 다뤄보자.