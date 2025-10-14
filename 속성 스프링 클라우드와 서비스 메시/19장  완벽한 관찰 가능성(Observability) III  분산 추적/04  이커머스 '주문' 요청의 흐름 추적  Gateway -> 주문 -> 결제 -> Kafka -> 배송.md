## 이커머스 '주문' 요청의 흐름 추적: Gateway -> 주문 -> 결제 -> Kafka -> 배송

지금까지 우리는 분산 추적의 모든 이론과 도구를 배웠습니다. 이제 이 모든 것을 하나로 엮어, 우리 이커머스 시스템에서 가장 복잡하고 중요한 비즈니스 트랜잭션인 **'주문 생성'** 요청의 전체 흐름을 처음부터 끝까지 추적해 보겠습니다.

이 시나리오에는 **동기식 REST 호출**과 **비동기 Kafka 메시징**이 모두 포함되어 있어, 분산 추적의 진정한 힘을 보여주는 완벽한 예시입니다.

**우리의 추적 스택:**
* **계측(Instrumentation):** 모든 마이크로서비스(`gateway`, `order`, `payment`, `shipping` 등)에 **OpenTelemetry Java 에이전트**를 연결하여 자동 계측합니다.
* **백엔드(Backend):** 로컬 Docker 환경에서 실행 중인 **Jaeger**.
* **컨텍스트 전파(Context Propagation):** HTTP 통신에는 **W3C Trace Context 헤더**가, Kafka 통신에는 **메시지 헤더**가 OTel 에이전트에 의해 자동으로 사용됩니다.

---

### 단일 주문 요청의 전체 생명주기

사용자가 '주문하기' 버튼을 누른 순간부터, Jaeger UI에 그려질 폭포(Waterfall) 차트를 상상하며 요청의 여정을 따라가 봅시다.

#### 1단계: 최초 진입 (API 게이트웨이)
* `POST /api/v1/orders` 요청이 `gateway-service`에 도착합니다.
* OTel 에이전트가 이 요청을 가로챕니다. 들어오는 요청에 `traceparent` 헤더가 없으므로, 새로운 **Trace를 시작**하고 고유한 `Trace ID`를 생성합니다. 그리고 이 요청 처리를 위한 **루트 스팬(Root Span)**을 만듭니다.
* **[Jaeger UI]** `Trace ID: abcdef...`에 대한 첫 번째 최상위 Span이 그려집니다.
    * `Span A: POST /api/v1/orders (service: gateway-service)`

#### 2단계: 동기 호출 → `order-service`
* 게이트웨이는 요청을 `order-service`로 라우팅합니다.
* OTel 에이전트는 나가는 HTTP 요청에 `traceparent` 헤더(`Trace ID`와 `Span A`의 ID 포함)를 **자동으로 주입**합니다.
* `order-service`가 요청을 받으면, OTel 에이전트는 `traceparent` 헤더를 읽어 **`Span A`의 자식 스팬**을 생성합니다.
* **[Jaeger UI]** `Span A` 아래에 들여쓰기된 자식 Span이 추가됩니다.
    * `Span B: POST /api/v1/orders (service: order-service)`

#### 3단계: 동기 호출 → `payment-service` (가정)
* `order-service`는 결제를 위해 `payment-service`를 동기식으로 호출합니다.
* 마찬가지로, OTel 에이전트가 컨텍스트를 전파하고 `payment-service`에서 `Span B`의 자식 스팬이 생성됩니다.
* **[Jaeger UI]** `Span B` 아래에 또 다른 자식 Span이 추가됩니다.
    * `Span C: POST /api/v1/payments (service: payment-service)`

#### 4단계: 비동기 발행 → Kafka
* `order-service`는 결제가 성공적으로 처리된 후, `OrderPaidEvent`를 Kafka로 발행합니다.
* OTel Kafka 계측 라이브러리는 `KafkaTemplate`이나 `StreamBridge`의 `send` 동작을 가로채, **Kafka 메시지의 헤더(Header)**에 `traceparent` 컨텍스트를 **자동으로 주입**합니다.
* 이벤트를 발행한 후, `order-service`는 사용자에게 "주문 완료" 응답을 보냅니다. 이 시점에서 `Span C`, `Span B`, `Span A`가 차례대로 종료됩니다. (사용자가 인지하는 응답 시간)

#### 5단계: 비동기 구독 → `shipping-service`
* 잠시 후(수 밀리초 또는 그 이상), `shipping-service`의 Kafka 컨슈머가 `OrderPaidEvent` 메시지를 수신합니다.
* OTel Kafka 계측 라이브러리는 메시지 헤더에서 `traceparent` 컨텍스트를 **자동으로 추출**합니다.
* 이 컨텍스트 정보를 사용하여, 이 메시지 처리를 위한 **새로운 Span**을 생성합니다. 이 Span은 **여전히 동일한 `Trace ID (abcdef...)`**를 가지며, 이벤트를 발행했던 `Span B`와의 관계를 인지합니다. (Jaeger에서는 'Follows From' 관계로 표시될 수 있습니다.)
* **[Jaeger UI]** 동일한 Trace 타임라인 위에서, `Span B`가 끝난 약간의 시간차를 두고 새로운 Span이 나타납니다.
    * `Span D: process OrderPaidEvent (service: shipping-service)`

---
### 최종 결과: 하나의 Trace, 완전한 이야기



Jaeger UI에서 `Trace ID: abcdef...`를 조회하면, 우리는 더 이상 파편화된 로그 조각이 아닌, **하나의 완벽한 이야기**를 보게 됩니다.
* 사용자가 경험한 **동기적 요청(Synchronous Path)**의 전체 시간과 각 서비스에서의 병목을 명확히 볼 수 있습니다. (Span A, B, C)
* 그리고 이 요청으로 인해 트리거된 **비동기 백그라운드 작업(Asynchronous Path)**이 언제 시작되어 얼마나 걸렸는지도 동일한 타임라인 상에서 확인할 수 있습니다. (Span D)

이것으로 19장을 마칩니다. 우리는 **로깅(무엇), 메트릭(얼마나), 그리고 추적(왜, 어디서)**이라는 관찰 가능성의 세 기둥을 모두 완성했습니다. 이 세 가지를 결합함으로써, 우리는 복잡한 MSA 내부에서 발생하는 어떤 문제라도 근본 원인을 신속하게 파악하고 해결할 수 있는 강력한 '투시경'을 갖게 되었습니다. 이제 이 관찰 가능한 시스템을 자동화된 파이프라인을 통해 배포하는 방법을 다음 20장에서 알아보겠습니다.