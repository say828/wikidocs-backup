## SAGA 구현 방식 2: 오케스트레이션 (Orchestration) - 상태 관리자를 둔 중앙 제어

SAGA 패턴을 구현하는 두 번째 방식은 **오케스트레이션(Orchestration, 지휘)**입니다.

이는 '교향악단'에 비유할 수 있습니다. 각 '연주자(마이크로서비스)'는 다른 연주자를 쳐다보지 않습니다. 대신 모든 연주자는 중앙의 **'지휘자(Orchestrator)'**의 지휘봉만 바라봅니다. 지휘자는 각 연주자에게 언제, 무엇을 연주할지 **'명령(Command)'**을 내리고, 연주가 끝나면 보고(이벤트)를 받아 다음 악장을 진행합니다. 실수가 발생하면 지휘자가 직접 나서서 바로잡습니다.

---

### 오케스트레이션 SAGA의 동작 흐름

이 방식에서는 **SAGA 오케스트레이터(Saga Orchestrator)** 또는 **SAGA 실행 조정자(SEC)**라는 새로운 책임 컴포넌트가 등장합니다. 이 오케스트레이터는 전체 비즈니스 트랜잭션의 상태를 관리하고, 각 참여 서비스에게 '다음에 할 일'을 명령(Command)으로 지시합니다.

#### 성공 시나리오 (Happy Path)


1.  **[Order Service]** 최초 주문 요청을 받으면, `Order`를 생성하는 대신 **`OrderSagaOrchestrator`**를 생성하고 시작합니다. 오케스트레이터는 SAGA의 현재 상태(예: `PENDING`)를 내부적으로 관리합니다.
2.  **[Orchestrator]** SAGA의 첫 단계로, `payment-service`에게 **`ProcessPaymentCommand`**를 보냅니다.
3.  **[Payment Service]** Command를 받아 결제를 처리하고, 성공했다는 의미로 **`PaymentCompletedEvent`**를 발행합니다. (Payment Service는 다음에 재고를 차감해야 한다는 사실을 전혀 모릅니다.)
4.  **[Orchestrator]** `PaymentCompletedEvent`를 구독하고 있다가, SAGA의 상태를 `PAYMENT_COMPLETED`로 변경합니다. 그리고 다음 단계로, `product-service`에게 **`DecreaseStockCommand`**를 보냅니다.
5.  **[Product Service]** Command를 받아 재고를 차감하고, 성공했다는 의미로 **`StockDecreasedEvent`**를 발행합니다.
6.  **[Orchestrator]** `StockDecreasedEvent`를 구독하고, 모든 단계가 성공했음을 인지합니다. SAGA의 상태를 `COMPLETED`로 변경하고, `order-service`에게 **`ConfirmOrderCommand`**를 보내 주문 상태를 최종 확정합니다.

#### 실패 및 보상 시나리오

'재고 차감' 실패 시, 오케스트레이터가 직접 보상 트랜잭션을 지휘합니다.

1.  ... (1~4번 단계는 성공) ...
2.  **[Product Service]** `DecreaseStockCommand`를 받았지만 실패하고, **`StockDecreaseFailedEvent`**를 발행합니다.
3.  **[Orchestrator]** `StockDecreaseFailedEvent`를 구독하고, SAGA가 실패했음을 인지합니다. SAGA 상태를 `ROLLING_BACK`으로 변경하고, **보상 트랜잭션을 역순으로 지휘하기 시작합니다.**
4.  **[Orchestrator]** `payment-service`에게 **`RefundPaymentCommand`**를 보냅니다.
5.  **[Orchestrator]** `payment-service`로부터 `PaymentRefundedEvent`를 받으면, `order-service`에게 **`CancelOrderCommand`**를 보냅니다.
6.  모든 보상 트랜잭션이 완료되면, SAGA 상태를 `FAILED`로 최종 변경합니다.

---

### 오케스트레이션 SAGA의 장점과 단점

#### 👍 장점

* **중앙 집중화된 로직과 명확한 가시성 (Centralized Logic & Visibility):**
    전체 비즈니스 프로세스가 **오케스트레이터 한 곳에 명시적으로 정의**됩니다. 덕분에 SAGA의 현재 상태를 추적하고 디버깅하기가 매우 쉽습니다. "이 주문은 왜 실패했지?"라는 질문에, 오케스트레이터의 상태만 확인하면 즉시 답을 얻을 수 있습니다.

* **순환 의존성 부재 (No Cyclic Dependencies):**
    참여 서비스들은 서로의 이벤트를 구독할 필요가 없습니다. 오직 오케스트레이터의 '명령'에 응답하고, 자신의 결과를 '이벤트'로 보고하기만 하면 됩니다. 서비스 간의 의존성이 단순해집니다.

* **복잡한 흐름 관리 용이 (Easier to Manage Complexity):**
    "만약 A 조건이면 C를 호출하고, B 조건이면 D를 호출하라"와 같은 복잡한 분기, 병렬 실행, 타임아웃 처리 등을 오케스트레이터 내부에서 상태 머신(State Machine)으로 쉽게 구현할 수 있습니다.

#### 👎 단점

* **과도한 책임 집중 (Risk of "God" Orchestrator):**
    오케스트레이터가 너무 많은 비즈니스 로직과 각 서비스의 내부 사정을 알게 되면, 그 자체가 또 다른 형태의 '중앙집중식 모놀리스'가 되어 서비스 간의 결합도를 높일 수 있습니다.

* **단일 실패 지점 (Single Point of Failure):**
    오케스트레이터 자체가 다운되면 전체 비즈니스 트랜잭션이 중단될 수 있습니다. (물론 오케스트레이터도 여러 인스턴스로 구성하여 고가용성을 확보해야 합니다.)

* **추가 인프라 비용:**
    오케스트레이터라는 새로운 컴포넌트를 설계하고, 구현하고, 운영해야 하는 부담이 있습니다.

---

**결론적으로,** 오케스트레이션 SAGA는 참여자가 많고, 조건부 로직이 포함된 **복잡하고 긴 비즈니스 트랜잭션**에 더 적합한 방식입니다. 코레오그래피의 '추적 불가'라는 단점을 해결하는 강력한 대안입니다.

일반적으로 SAGA 오케스트레이터는 트랜잭션을 시작하는 서비스(우리 예제에서는 `order-service`) 내부에 상태 머신으로 구현하거나, `Camunda`나 `AWS Step Functions` 같은 전문 워크플로우 엔진을 사용하기도 합니다.

다음 절에서는 이 두 방식 중, 상대적으로 구현이 간단한 **코레오그래피 기반 SAGA**를 실제 코드로 구현하여 '주문' 프로세스를 완성해 보겠습니다.