## Grafana/Prometheus 연동: Envoy 프록시에서 노출하는 L7 메트릭 수집

18장에서 우리는 `Micrometer`와 `Spring Boot Actuator`를 사용하여 애플리케이션 \*\*'내부'\*\*에서 메트릭을 생성하고, Prometheus가 각 애플리케이션의 `/actuator/prometheus` 엔드포인트를 수집(Scrape)하도록 설정했습니다.

하지만 분산 추적과 마찬가지로, Istio 서비스 메시는 메트릭 수집 또한 **플랫폼 레벨로 외부화**할 수 있는 강력한 기능을 제공합니다. 모든 트래픽을 가로채는 Envoy 사이드카 프록시는, 자신이 처리하는 트래픽에 대한 매우 상세하고 표준화된 메트릭을 생성할 수 있는 가장 이상적인 위치이기 때문입니다.

-----

### 플랫폼 레벨 메트릭의 장점

애플리케이션이 아닌 Envoy 프록시가 메트릭을 생성하면 어떤 이점이 있을까요?

1.  **언어 독립성 및 일관성:** `istio_requests_total`이라는 표준 메트릭은 서비스가 코틀린으로 작성되었든, 파이썬으로 작성되었든 **항상 동일한 이름과 레이블 구조**를 가집니다. 이를 통해 전체 MSA에 대한 일관된 대시보드와 경고(Alert) 규칙을 손쉽게 만들 수 있습니다.
2.  **애플리케이션 부담 제로:** 개발자는 더 이상 `Micrometer` 같은 메트릭 라이브러리를 추가하거나, HTTP 요청을 계측하는 코드에 대해 신경 쓸 필요가 없습니다. 플랫폼이 모든 서비스에 대해 풍부한 '골든 시그널(Golden Signals)' 메트릭을 **무료로** 제공합니다.
3.  **풍부한 네트워크 레벨 정보:** Envoy는 애플리케이션 레벨에서는 알 수 없는 상세한 네트워크 정보를 메트릭 레이블로 제공합니다. 예를 들어, `response_flags` 레이블을 통해 "업스트림 연결 실패"(`UC`)인지, "서킷 브레이커에 의해 차단됨"(`UO`)인지 등을 정확히 구분할 수 있습니다.

-----

### Istio 표준 메트릭과 강력한 레이블

Istio는 기본적으로 다음과 같은 매우 유용한 표준 L7 메트릭을 제공합니다.

  * `istio_requests_total`: 서비스 간 HTTP/gRPC 요청의 총 횟수 (Counter)
  * `istio_request_duration_milliseconds`: 요청 처리 시간 분포 (Histogram)

이 메트릭들의 진정한 힘은, 다음과 같이 자동으로 추가되는 풍부한 \*\*레이블(Label)\*\*에 있습니다.

  * **`source_app`, `destination_app`**: 어떤 서비스가 어떤 서비스를 호출했는가?
  * **`source_version`, `destination_version`**: 호출한 서비스와 호출된 서비스의 버전은 무엇인가?
  * **`response_code`**: HTTP 응답 코드는 무엇인가? (200, 404, 503 등)
  * **`response_flags`**: 네트워크 레벨의 통신 결과는 어떠한가?

이 레이블 덕분에, 우리는 Grafana에서 다음과 같은 매우 강력하고 정교한 PromQL 쿼리를 작성할 수 있습니다.

```promql
# order-service v1이 product-service v2를 호출하여 503 에러가 발생한 요청의 1분당 비율
sum(rate(istio_requests_total{
  source_app="order-service",
  source_version="v1",
  destination_app="product-service",
  destination_version="v2",
  response_code="503"
}[1m]))
```

-----

### 하이브리드 접근법: Istio 메트릭 + Actuator 메트릭

"그렇다면 이제 Spring Boot Actuator는 필요 없나요?" 아닙니다. 분산 추적과 마찬가지로, 메트릭 또한 **하이브리드(Hybrid)** 접근 방식이 가장 이상적입니다.

1.  **Istio/Envoy:** \*\*서비스 간 통신(네트워크 트래픽)\*\*에 대한 전문가입니다.

      * `요청 수`, `응답 시간`, `에러율`과 같은 모든 표준 '골든 시그널' 메트릭은 Istio를 통해 수집하는 것이 가장 일관되고 정확합니다.

2.  **Spring Boot Actuator:** **애플리케이션 '내부' 상태**에 대한 전문가입니다.

      * **JVM 메트릭:** `jvm_memory_used_bytes`, `jvm_gc_pause_seconds` 등 JVM의 상세 상태는 Actuator만이 알 수 있습니다.
      * **비즈니스 메트릭:** `orders_created_total`과 같이 우리가 18장에서 직접 만든 커스텀 비즈니스 메트릭은 애플리케이션 내부에서만 생성할 수 있습니다.
      * **의존성 내부 메트릭:** `HikariCP` 커넥션 풀의 상태와 같은 라이브러리 내부 메트릭.

**최종 아키텍처:**

1.  Prometheus가 클러스터 내의 **모든 Envoy 사이드카의 `/stats/prometheus` 엔드포인트**를 수집하도록 설정합니다. (Istio 설치 시 자동 설정)
2.  Prometheus가 **모든 애플리케이션 Pod의 `/actuator/prometheus` 엔드포인트** 또한 수집하도록 추가로 설정합니다.
3.  Grafana 대시보드에서는 이 두 가지 소스의 데이터를 모두 활용하여, 서비스 간 트래픽 현황(from Istio)과 JVM 상태(from Actuator), 그리고 비즈니스 KPI(from Actuator)를 **하나의 화면에서 통합적으로** 모니터링합니다.

이것으로 23장을 마칩니다. 우리는 Istio의 강력한 보안 기능(`PeerAuthentication`, `AuthorizationPolicy`)과 관찰 가능성 기능(Tracing, Metrics)을 모두 탐구했습니다. Istio가 제공하는 플랫폼 레벨의 텔레메트리와 애플리케이션 레벨의 텔레메트리를 결합함으로써, 우리는 시스템의 동작과 보안 상태를 완벽하게 파악하고 제어할 수 있는 진정한 의미의 '관찰 가능한 시스템'을 완성했습니다.