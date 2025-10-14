## OpenTelemetry 표준의 이해: Trace, Span, Context Propagation

분산 추적의 '필요성'을 이해했다면, 이제 '어떻게' 구현할 것인가의 문제에 직면합니다. 과거에는 `Jaeger`, `Zipkin`, `Datadog` 등 각 모니터링 솔루션 업체들이 자신들만의 독자적인 방식으로 데이터를 수집하고 전송했습니다. 이는 특정 벤더의 코드에 종속되는 **'벤더 종속성(Vendor Lock-in)'** 문제를 야기했습니다.

만약 `Zipkin`의 라이브러리를 사용하여 코드를 작성했는데, 나중에 `Datadog`으로 모니터링 시스템을 교체하려면, 애플리케이션의 모든 추적 관련 코드를 재작성해야 했습니다.

이 문제를 해결하기 위해 등장한 것이 바로 \*\*OpenTelemetry(OTel)\*\*입니다.

-----

### OpenTelemetry: 관찰 가능성을 위한 단일 표준

**OpenTelemetry**는 CNCF의 프로젝트로, \*\*추적(Traces), 메트릭(Metrics), 로그(Logs)\*\*와 같은 모든 원격 측정(Telemetry) 데이터를 수집하고 전송하는 것에 대한 **벤더 중립적인 단일 표준**을 제공합니다.

> **"OpenTelemetry는 관찰 가능성(Observability)을 위한 SLF4J이다."**

개발자는 `SLF4J`라는 표준 인터페이스에 맞춰 로그 코드를 작성하고, 실제 구현체는 `Logback`으로 갈아 끼울 수 있는 것처럼, OpenTelemetry의 **표준 API**에 맞춰 추적 코드를 **한 번만 작성**하면, **'내보내기(Exporter)'** 설정만 바꾸어 `Jaeger`, `Zipkin`, `Prometheus`, `Datadog` 등 원하는 어떤 모니터링 백엔드로든 데이터를 전송할 수 있습니다.

#### OpenTelemetry의 핵심 구성 요소

  * **API:** 개발자가 코드에서 직접 사용하는 벤더 중립적인 인터페이스입니다. (예: `Tracer.spanBuilder().startSpan()`)
  * **SDK (Software Development Kit):** API에 대한 공식 구현체입니다. 샘플링(Sampling) 정책이나 Exporter 설정 등을 여기서 구성합니다.
  * **Exporter (내보내기):** SDK가 수집한 데이터를 특정 백엔드(Jaeger, Zipkin 등)의 형식에 맞게 변환하여 전송하는 플러그인입니다.
  * **Collector (수집기, 선택 사항):** 여러 소스로부터 원격 측정 데이터를 받아, 일괄 처리(예: 공통 속성 추가, 필터링)한 뒤, 하나 이상의 백엔드로 내보내는 독립적인 프록시 서비스입니다.

-----

### OpenTelemetry가 정의하는 Trace 모델

OpenTelemetry는 19장 00절에서 소개한 개념들을 다음과 같이 표준화합니다.

  * **Trace:** 요청의 전체 흐름을 나타내는 Span들의 집합입니다.

  * **Span:** 작업의 기본 단위입니다. Span은 다음과 같은 중요한 정보를 가집니다.

      * **이름:** "HTTP GET /api/v1/products/{productId}" 와 같은 작업 이름.
      * **부모 Span ID:** Trace의 나무 구조를 형성하는 연결 고리.
      * **타임스탬프:** 시작 시간과 종료 시간 (이를 통해 '소요 시간' 계산).
      * **속성 (Attributes):** 작업을 설명하는 Key-Value 쌍의 풍부한 메타데이터. (예: `http.method="GET"`, `db.statement="SELECT * ..."`)
      * **이벤트 (Events):** Span 내에서 발생한 특정 시점의 로그. (예: "Cache miss at 10:05.123")
      * **상태 (Status):** 작업의 성공(`OK`) 또는 실패(`ERROR`) 여부.

  * **컨텍스트 전파 (Context Propagation):**
    서비스 경계를 넘어 Span들을 연결하는 메커니즘입니다. OpenTelemetry는 **W3C Trace Context**라는 웹 표준을 기본으로 사용합니다.

      * `order-service`가 `product-service`를 호출할 때, `traceparent`라는 HTTP 헤더에 `Trace ID`와 부모 `Span ID`를 담아 보냅니다.
      * `product-service` 측의 OpenTelemetry SDK는 이 헤더를 자동으로 인식하여, 새로 생성하는 Span이 올바른 부모를 갖도록 Trace 트리를 이어나갑니다.

-----

### 자동 계측 (Auto-Instrumentation)의 마법

OpenTelemetry의 가장 강력한 기능 중 하나는 \*\*'자동 계측'\*\*입니다.

우리는 **애플리케이션 코드를 단 한 줄도 수정하지 않고**, 애플리케이션 실행 시 \*\*Java 에이전트(Agent)\*\*를 붙여주는 것만으로, OpenTelemetry가 수많은 작업을 자동으로 추적하고 Span을 생성하게 할 수 있습니다.

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -jar my-application.jar
```

이 에이전트는 바이트코드 조작(Bytecode Instrumentation) 기술을 사용하여, 다음과 같은 잘 알려진 라이브러리/프레임워크의 코드 실행을 런타임에 가로채 Span을 자동으로 생성하고 컨텍스트를 전파합니다.

  * Spring MVC / WebFlux (HTTP 서버 요청)
  * `WebClient`, `RestTemplate`, OpenFeign (HTTP 클라이언트 요청)
  * JDBC, R2DBC (DB 쿼리)
  * Kafka Producer / Consumer
  * Redis (Lettuce) 등...

**결론적으로,** OpenTelemetry는 클라우드 네이티브 애플리케이션의 데이터 수집 방식을 표준화하여, 개발자를 특정 벤더 기술로부터 해방시켰습니다. 특히 강력한 '자동 계측' 기능을 통해, 우리는 최소한의 노력으로 우리 MSA 시스템 내부 동작에 대한 깊은 가시성을 확보할 수 있습니다. 다음 절에서는 이 OpenTelemetry를 우리 스프링 부트 애플리케이션에 적용하는 방법을 알아보겠습니다.