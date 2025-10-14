## 트랜잭셔널 아웃박스(Transactional Outbox) 패턴: 이벤트 발행과 DB 트랜잭션 원자성 보장

우리는 이벤트를 통해 데이터를 동기화하는 강력한 모델을 구축했습니다. 하지만 여전히 해결되지 않은 미묘하고 치명적인 문제가 하나 남아있습니다.

> **"DB 트랜잭션 `COMMIT`과 Kafka로의 `이벤트 발행`이라는 두 가지 작업을 어떻게 원자적(Atomic)으로 묶을 수 있을까?"**

`OrderService`가 주문을 처리하는 과정을 생각해 봅시다.

1.  `@Transactional` 메서드 안에서 `orderRepository.save(order)`를 호출합니다.
2.  같은 메서드 안에서 `kafkaTemplate.send(...)`를 호출하여 이벤트를 발행합니다.

이 두 작업은 별개의 시스템(RDBMS, Kafka)에 대한 호출이므로, 단일 트랜잭션으로 묶일 수 없습니다. 따라서 다음과 같은 실패 시나리오가 발생할 수 있습니다.

  * **시나리오 A:** DB `COMMIT`은 성공했는데, `kafkaTemplate.send()`를 호출하기 직전에 애플리케이션이 다운된다.
      * **결과:** 주문은 생성되었지만, 이벤트는 **영원히 유실**됩니다. `shipping-service`는 이 주문의 존재를 알지 못합니다. (**가장 흔하고 위험한 문제**)
  * **시나리오 B:** `kafkaTemplate.send()`는 성공했는데, 그 직후 DB `COMMIT`이 제약 조건 위반 등으로 실패하고 롤백된다.
      * **결과:** 실제로는 존재하지 않는 주문에 대한 이벤트가 발행됩니다. (**유령 이벤트**)

이러한 데이터 불일치는 MSA에서 가장 잡기 어려운 버그의 원인이 됩니다.

-----

### 해답: 트랜잭셔널 아웃박스 (Transactional Outbox) 패턴

이 문제를 해결하는 가장 표준적이고 견고한 패턴이 바로 **트랜잭셔널 아웃박스**입니다.

**핵심 아이디어:** "두 개의 외부 시스템(DB, Kafka)에 동시에 쓰려고 시도하지 마라. 대신, 우리가 100% 제어할 수 있는 **단 하나의 로컬 데이터베이스에만 원자적으로 모두 써라.**"

**우체통 비유 📮:**

  * **기존 방식:** 중요한 편지(비즈니스 데이터)를 작성하고, 동시에 우체국(Kafka)으로 달려가 편지(이벤트)를 부치려고 합니다. 만약 우체국으로 가는 길에 넘어지면(애플리케이션 다운), 편지는 전달되지 못합니다.
  * **아웃박스 방식:** 중요한 편지를 작성하고, 그 편지를 복사하여 내 책상 위의 \*\*'발신함(Outbox)'\*\*에 넣어둡니다. 그리고 책상 서랍을 잠급니다(**DB `COMMIT`**). 이제 내 작업물과 '보낼 편지'는 **하나의 트랜잭션**으로 안전하게 보관되었습니다. 그 후, \*\*믿을 수 있는 집배원(별도의 프로세스)\*\*이 주기적으로 내 발신함을 확인하고, 편지를 수거하여 우체국에 **안전하게 전달**해 줍니다.

-----

### 아웃박스 패턴의 아키텍처

1.  **`outbox` 테이블 생성:**
    이벤트를 발행하는 서비스(예: `order-service`)의 **데이터베이스 내부에** `outbox`라는 특별한 테이블을 생성합니다.
    | id (UUID) | aggregate\_type | aggregate\_id | event\_type | payload (JSON) |
    | :--- | :--- | :--- | :--- | :--- |
    | `...` | `Order` | `123` | `OrderCreated` | `{"orderId":123, ...}`|

2.  **애플리케이션 로직 변경:**
    `OrderService`의 `@Transactional` 메서드는 이제 Kafka에 직접 이벤트를 보내는 대신, \*\*`outbox` 테이블에 '보낼 이벤트'를 `INSERT`\*\*합니다.

    ```kotlin
    @Service
    @Transactional
    class OrderService(
        private val orderRepository: OrderRepository,
        private val outboxRepository: OutboxRepository // 아웃박스 리포지토리
    ) {
        fun createOrder(command: CreateOrderCommand): Long {
            // 1. 비즈니스 데이터 저장
            val savedOrder = orderRepository.save(...)
            
            // 2. 발행할 이벤트를 Outbox 테이블에 저장
            val event = OutboxEvent(
                aggregateType = "Order",
                aggregateId = savedOrder.id!!,
                eventType = "OrderCreated",
                payload = toJson(OrderCreatedEvent(...))
            )
            outboxRepository.save(event)

            return savedOrder.id!!
            // 메서드가 종료되면, orders 테이블과 outbox 테이블의 변경사항이
            // '하나의 DB 트랜잭션'으로 원자적으로 COMMIT 됩니다.
        }
    }
    ```

    이제 `OrderService`는 Kafka의 존재 자체를 알 필요가 없어졌습니다\! 비즈니스 로직이 메시징 인프라로부터 완벽하게 분리되었습니다.

3.  **이벤트 전달자 (Message Relay): Debezium**
    그렇다면 `outbox` 테이블에 쌓인 이벤트를 누가 Kafka로 전달할까요? 바로 11장 03절에서 배운 \*\*Debezium(CDC)\*\*이 이 '믿을 수 있는 집배원' 역할을 수행합니다.

      * **Debezium CDC 커넥터**가 `order-service` DB의 **`outbox` 테이블을 바라보도록** 설정합니다.
      * `OrderService`의 트랜잭션이 커밋되어 `outbox` 테이블에 새로운 행이 `INSERT`되면, Debezium이 이 변경을 즉시 캡처합니다.
      * Debezium은 `outbox` 행의 `payload` 컬럼을 읽어, 이 내용을 실제 Kafka 토픽(예: `ecommerce.orders.created`)으로 발행합니다.
      * (선택) 메시지가 성공적으로 발행되면, `outbox` 테이블의 해당 행을 삭제하는 별도의 프로세스를 두어 테이블을 깨끗하게 유지할 수 있습니다.

-----

이 `트랜잭셔널 아웃박스` 패턴을 통해, 우리는 \*\*'최소 한 번 발행 보장(At-least-once delivery)'\*\*을 완벽하게 달성했습니다. DB 트랜잭션이 성공하면 이벤트는 **반드시** 발행되며, DB 트랜잭션이 실패하면 이벤트는 **절대** 발행되지 않습니다.

이것으로 11장을 마칩니다. 우리는 `Database per Service` 패턴을 채택하고, 그로 인해 발생하는 '데이터 일관성' 문제를 `최종 일관성`, `CDC`, 그리고 `트랜잭셔널 아웃박스`라는 강력한 패턴들로 해결했습니다. 이제 우리의 MSA는 대규모 분산 환경에서도 데이터의 정합성을 유지할 수 있는 견고한 기반을 갖추었습니다. 다음 12장에서는 코틀린의 꽃, **코루틴**을 활용하여 시스템의 성능을 한 차원 더 끌어올려 보겠습니다.