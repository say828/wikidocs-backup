## 이벤트 기반 아키텍처(EDA)란 무엇인가: 시스템 간의 결합도(Coupling) 낮추기

07장 00절에서 우리는 `order-service`가 `OrderPaidEvent`라는 '이벤트'를 '메시지 큐'에 던지는 아이디어를 접했습니다. 이 모델이 바로 \*\*이벤트 기반 아키텍처(Event-Driven Architecture, EDA)\*\*의 핵심입니다.

EDA는 시스템의 동작이 '명령(Command)'이 아닌 \*\*'이벤트(Event)'\*\*의 발생과 그에 대한 '반응(Reaction)'으로 구동되는 아키텍처 스타일입니다.

-----

### EDA의 4가지 핵심 구성 요소

1.  **이벤트 (Event):**

      * 시스템에서 발생한 **'의미 있는 사건'** 또는 \*\*'상태의 변경'\*\*을 나타내는 데이터 레코드입니다.
      * **과거 시제**로 명명하는 것이 관례입니다. (예: `PayOrder`는 명령, `OrderPaid`는 이벤트)
      * 이벤트는 \*\*'이미 일어난 사실(Fact)'\*\*이므로 \*\*불변(Immutable)\*\*입니다.
      * 예시: `OrderPaidEvent`, `MemberJoinedEvent`, `StockDecreasedEvent`

2.  **이벤트 생산자 (Event Producer / Publisher):**

      * 이벤트를 **생성하고 발행**하는 주체입니다.
      * 우리 예시에서는 `order-service`가 `OrderPaidEvent`의 생산자입니다.

3.  **이벤트 소비자 (Event Consumer / Subscriber):**

      * 특정 이벤트에 **관심을 갖고 구독**하여, 해당 이벤트가 발생했을 때 특정 동작을 수행하는 주체입니다.
      * `shipping-service`, `notification-service`가 `OrderPaidEvent`의 소비자입니다.

4.  **이벤트 채널 (Event Channel / Message Broker):**

      * 생산자가 발행한 이벤트를 소비자에게 전달하는 \*\*'중간 매개체'\*\*입니다. 이 채널이 있기에 생산자와 소비자는 서로를 몰라도 통신할 수 있습니다.
      * `Kafka`, `RabbitMQ` 등이 이 역할을 수행합니다.

-----

### EDA의 궁극적인 목표: 결합도(Coupling) 낮추기

EDA를 도입하는 가장 큰 이유는 시스템 간의 **결합도를 극단적으로 낮추기 위함**입니다.

#### 동기식 API 호출 (높은 결합도)

```
[Order Service] --- (HTTP Call) ---> [Shipping Service]
      |
      +------------ (HTTP Call) ---> [Notification Service]
      |
      +------------ (HTTP Call) ---> [Member Service]
```

  * **지식의 결합:** `order-service`는 `shipping-service`, `notification-service`, `member-service`의 **존재와 주소(API Endpoint)를 모두 알고 있어야 합니다.** 만약 새로운 `data-analytics-service`가 '주문 완료' 정보를 필요로 하게 되면, `order-service`의 **코드를 직접 수정하고 재배포**해야 합니다.
  * **시간적 결합:** 모든 서비스가 **동시에** 정상적으로 동작해야만 전체 로직이 성공합니다.

#### 이벤트 기반 아키텍처 (낮은 결합도)

```
                          +-----------------> [Shipping Service]
                          |
[Order Service] --(Event)--> [Message Broker] --+-----------------> [Notification Service]
                          |
                          +-----------------> [Member Service]
                          |
                          +-----------------> [Data Analytics Service]
```

  * **지식의 결합 해소 (Producer Ignorance):** `order-service`는 오직 '메시지 브로커'의 존재만 알면 됩니다. `shipping-service`나 `notification-service`가 존재하는지, 몇 대가 실행 중인지, 심지어 **누가 자신의 이벤트를 구독하는지 전혀 알 필요가 없습니다.**
  * **새로운 기능 추가의 유연성:** `data-analytics-service`를 새로 추가하고 싶다면, `order-service`의 코드는 **단 한 줄도 건드리지 않고**, 새로운 서비스가 메시지 브로커의 `OrderPaidEvent`를 구독하도록 만들기만 하면 됩니다. 시스템이 훨씬 유연하고 확장 가능해집니다.
  * **시간적 결합 해소:** `shipping-service`가 다운되어 있어도, `order-service`는 아무 문제 없이 이벤트를 발행할 수 있습니다. 이벤트는 메시지 브로커에 안전하게 보관되었다가, `shipping-service`가 복구되었을 때 전달됩니다.

-----

이처럼 EDA는 각 마이크로서비스를 **'독립적으로 진화할 수 있는'** 진정한 자율적인 컴포넌트로 만들어주는 강력한 패러다임입니다. 이는 MSA의 철학과 완벽하게 일치합니다.

다음 절에서는 이 EDA를 구현하기 위한 핵심 기술, 즉 '이벤트 채널' 역할을 하는 메시지 큐 `Kafka`와 `RabbitMQ`를 비교하고, 어떤 상황에 어떤 기술을 선택해야 하는지 알아보겠습니다.