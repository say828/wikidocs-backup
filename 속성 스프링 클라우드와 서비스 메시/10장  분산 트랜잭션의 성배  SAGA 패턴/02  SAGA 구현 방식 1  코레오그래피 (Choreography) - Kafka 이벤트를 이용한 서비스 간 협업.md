## SAGA 구현 방식 1: 코레오그래피 (Choreography) - Kafka 이벤트를 이용한 서비스 간 협업

SAGA 패턴을 구현하는 첫 번째 방식은 **코레오그래피(Choreography, 안무)**입니다.

이름처럼, 중앙의 '지휘자(Orchestrator)'가 없이 각각의 '무용수(마이크로서비스)'가 서로의 '춤 동작(이벤트)'을 보고 다음에 무엇을 할지 스스로 결정하여 협업하는 방식입니다. 모든 통신은 08장에서 구축한 **Kafka**와 같은 메시지 브로커를 통해 비동기 이벤트로 이루어집니다.

---

### 코레오그래피 SAGA의 동작 흐름

각 서비스는 자신의 로컬 트랜잭션을 완료한 후, "내가 내 일을 끝냈다"는 사실을 담은 이벤트를 발행합니다. 그러면 다음 단계의 책임을 가진 서비스가 이 이벤트를 구독하여 자신의 작업을 시작하는 연쇄 반응이 일어납니다.

#### 성공 시나리오 (Happy Path)


1.  **[Order Service]** 주문을 생성하고 `COMMIT`한 뒤, `OrderCreatedEvent`를 Kafka로 발행합니다.
2.  **[Payment Service]** `OrderCreatedEvent`를 구독하여 결제를 처리하고 `COMMIT`한 뒤, `PaymentCompletedEvent`를 발행합니다.
3.  **[Product Service]** `PaymentCompletedEvent`를 구독하여 재고를 차감하고 `COMMIT`한 뒤, `StockDecreasedEvent`를 발행합니다.
4.  **[Order Service]** `StockDecreasedEvent`를 다시 구독하여, 최종적으로 주문 상태를 `CONFIRMED`로 변경하고 `COMMIT`합니다.

#### 실패 및 보상 시나리오

'재고 차감'이 실패하는 경우, 각 서비스는 '성공 이벤트'뿐만 아니라 **'실패 이벤트'**도 구독하고 있어야 합니다.

1.  ... (1, 2번 단계는 성공) ...
2.  **[Product Service]** `PaymentCompletedEvent`를 구독했지만, 재고 부족으로 로컬 트랜잭션이 **실패**합니다. 그리고 `StockDecreaseFailedEvent`를 발행합니다.
3.  **[Payment Service]** `StockDecreaseFailedEvent`를 구독하고 있다가, 이 이벤트를 받고 자신의 **보상 트랜잭션(결제 환불)**을 실행합니다. 그 후 `PaymentRefundedEvent`를 발행할 수 있습니다.
4.  **[Order Service]** `StockDecreaseFailedEvent`를 구독하고 있다가, 이 이벤트를 받고 자신의 **보상 트랜잭션(주문 취소)**을 실행합니다.

---

### 코레오그래피 SAGA의 장점과 단점

#### 👍 장점

* **단순함과 느슨한 결합 (Simplicity & Loose Coupling):**
    중앙 관리 컴포넌트가 없으므로 구조가 단순하고, 단일 실패 지점(SPOF)이 없습니다. 각 서비스는 오직 메시지 브로커하고만 통신하면 되므로, 서비스 간의 결합도가 매우 낮습니다.

* **쉬운 기능 추가 (Easy to add new services):**
    '주문 완료 시 이메일 발송' 기능을 추가하고 싶다면, 기존 `order-service`나 `payment-service`의 코드를 전혀 건드리지 않고, `PaymentCompletedEvent`를 구독하는 새로운 `notification-service`를 만들기만 하면 됩니다. 매우 유연하고 확장성이 좋습니다.

#### 👎 단점

* **전체 흐름 추적 및 디버깅의 어려움 (Difficulty in Tracking & Debugging):**
    이것이 코레오그래피의 가장 큰 문제입니다. 전체 비즈니스 트랜잭션의 로직이 여러 서비스의 '이벤트 구독' 코드 속에 흩어져 있습니다. "주문이 왜 실패했지?"라는 질문에 답하기 위해, `order-service`, `payment-service`, `product-service`의 로그를 모두 뒤져가며 이벤트의 흐름을 **수동으로 재구성**해야 합니다. SAGA에 참여하는 서비스가 4~5개를 넘어가면 이 추적은 거의 불가능에 가까워집니다.

* **순환 의존성 발생 가능성 (Potential for Cyclic Dependencies):**
    서비스 A가 발행한 이벤트를 B가 구독하고, B가 발행한 이벤트를 다시 A가 구독하는 등의 순환 의존성이 발생하기 쉬우며, 이를 파악하기 어렵습니다.

* **계약 관리의 복잡성:**
    어떤 서비스가 어떤 이벤트를 발행하고 구독하는지에 대한 '계약'이 명시적으로 관리되지 않으면, 시스템 전체의 동작을 이해하기 매우 어렵습니다.

---

**결론적으로,** 코레오그래피 SAGA는 참여하는 서비스가 2~3개 정도로 적고, 비즈니스 흐름이 단순한 **'선형적(Linear)' 프로세스**에 매우 적합합니다.

하지만 "만약 결제 수단이 '쿠폰'이면 재고 확인을 건너뛰고, '해외 배송'이면 통관 서비스를 먼저 호출해야 한다"와 같이 복잡한 **'조건부 분기'**가 포함된 비즈니스 트랜잭션이라면, 코레오그래피 방식의 복잡성은 기하급수적으로 증가합니다. 이 문제를 해결하기 위한 대안이 바로 다음 절에서 다룰 **'오케스트레이션(Orchestration)'** 방식입니다.