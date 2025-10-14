## 얼럿(Alerting): Alertmanager를 이용한 장애 상황 자동 알림

18장 03절에서 우리는 Grafana를 통해 시스템의 모든 건강 지표를 한눈에 볼 수 있는 아름다운 대시보드를 구축했습니다. 하지만 대시보드는 우리가 \*\*'직접 보고 있을 때'\*\*만 의미가 있습니다. 개발자와 운영자가 24시간 내내 대시보드만 쳐다볼 수는 없습니다.

시스템에 문제가 발생했을 때, 우리가 시스템을 찾아가는 것이 아니라 **시스템이 우리에게 '문제가 발생했다'고 능동적으로 알려줘야 합니다.** 이것이 바로 \*\*얼럿(Alerting, 경고)\*\*의 역할입니다.

모니터링이 '관찰'이라면, 얼럿은 '자동화된 비상벨'입니다.

-----

### Prometheus + Alertmanager: 똑똑한 경고 시스템

프로메테우스 생태계는 이 얼럿 기능을 **Prometheus Server**와 **Alertmanager**라는 두 컴포넌트의 협업을 통해 구현합니다.

1.  **Prometheus Server (경고 규칙 평가):**

      * 프로메테우스 서버는 주기적으로 **'경고 규칙(Alerting Rules)'** 파일을 읽어들입니다.
      * 이 규칙은 "만약 이 PromQL 쿼리의 결과가 5분 이상 참(true)이면, 경고를 발생시켜라"와 같은 형식으로 작성됩니다.
      * 조건이 충족되면, 프로메테우스는 해당 경고를 'FIRING' 상태로 만들고, 이 경고 정보를 **Alertmanager**로 보냅니다.

2.  **Alertmanager (알림 발송 및 관리):**

      * Alertmanager는 프로메테우스로부터 경고를 전달받아, 이를 실제 '알림(Notification)'으로 변환하고 발송하는 전문 컴포넌트입니다.
      * Alertmanager는 단순 전달을 넘어, 다음과 같은 지능적인 역할을 수행합니다.
          * **중복 제거 (Deduplication):** 10대의 `product-service` Pod가 동시에 다운되어 10개의 경고가 발생해도, 이를 하나의 알림으로 묶어서 보냅니다.
          * **그룹핑 (Grouping):** 연관된 여러 경고(예: 'CPU 사용량 높음', 'API 응답 시간 지연')를 하나의 알림으로 그룹화하여 보냅니다.
          * **라우팅 (Routing):** 경고의 '심각도(severity)'나 '팀(team)' 레이블에 따라, 알림을 보낼 채널을 다르게 지정할 수 있습니다. (예: `severity=critical` 경고는 **PagerDuty**로, `team=payment` 경고는 **결제팀 Slack 채널**로 전송)
          * **소거 (Silencing):** 예정된 배포 작업 시간 동안 발생하는 예상된 경고를 일시적으로 음소거할 수 있습니다.

-----

### 실전 예제: 이커머스 시스템을 위한 경고 규칙 작성

`prometheus-rules.yml`과 같은 파일을 생성하여, 우리 시스템에 필수적인 경고 규칙들을 PromQL로 정의해 봅시다.

```yaml
# prometheus-rules.yml
groups:
- name: ecommerce-alerts
  rules:
  # --- 규칙 1: 서비스 인스턴스 다운 감지 ---
  - alert: InstanceDown
    expr: up{job="kubernetes-pods"} == 0
    for: 1m # 1분 이상 down 상태가 지속되면
    labels:
      severity: critical
    annotations:
      summary: "Instance {{ $labels.pod }} down"
      description: "{{ $labels.pod }} of job {{ $labels.job }} has been down for more than 1 minute."

  # --- 규칙 2: API 5xx 에러 비율 급증 감지 ---
  - alert: HighErrorRate
    # 5분간 5xx 에러 비율이 5%를 초과하면
    expr: (sum(rate(http_server_requests_seconds_count{status=~"5.*"}[5m])) by (application) / sum(rate(http_server_requests_seconds_count[5m])) by (application)) * 100 > 5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High HTTP 5xx error rate on {{ $labels.application }}"
      description: "{{ $labels.application }} has an error rate above 5% for the last 5 minutes."

  # --- 규칙 3: 비즈니스 메트릭 기반 이상 감지 (매우 중요!) ---
  - alert: NoOrdersPlaced
    # 10분 동안 주문 생성 메트릭(orders_created_total)의 증가율이 0이면
    expr: rate(orders_created_total[10m]) == 0
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "No orders have been placed in the last 10 minutes"
      description: "The system might be experiencing a critical issue as no orders are being processed."
```

  * 이 규칙 파일을 프로메테우스 설정에 추가하면, 프로메테우스는 이 규칙들을 주기적으로 평가하기 시작합니다.

### 최종 알림 흐름

1.  `order-service`에 치명적인 버그가 발생하여 10분 동안 주문이 한 건도 생성되지 않습니다.
2.  프로메테우스는 `orders_created_total` 메트릭을 계속 수집하지만, 그 값이 증가하지 않는 것을 발견합니다.
3.  10분이 지나자 `NoOrdersPlaced` 경고 규칙(`expr`)이 참이 되고, 프로메테우스는 `FIRING` 상태의 경고를 Alertmanager로 보냅니다.
4.  Alertmanager는 이 경고의 `severity: critical` 레이블을 보고, 미리 설정된 라우팅 규칙에 따라 **'개발팀 전체' Slack 채널**과 **'SRE팀' PagerDuty**로 즉시 알림을 보냅니다.
5.  개발팀은 "🔥 **FIRING [CRITICAL]: No orders have been placed in the last 10 minutes**" 라는 메시지를 받고, 사용자가 문제를 인지하기 전에 신속하게 대응을 시작합니다.

이것으로 18장을 마칩니다. 모니터링은 단순히 Grafana 대시보드를 아름답게 꾸미는 것이 아니라, **프로메테우스의 경고 규칙과 Alertmanager의 자동화된 알림**을 통해 시스템의 문제를 능동적으로 감지하고 신속하게 대응하는 \*\*'면역 체계'\*\*를 구축하는 것입니다.

이제 우리는 시스템의 로그(무슨 일이 있었나?)와 메트릭(전반적인 건강 상태는 어떤가?)을 모두 확보했습니다. 하지만 아직 풀리지 않은 마지막 질문이 있습니다. "API 응답이 느려졌는데, 그 시간은 **정확히 어떤 서비스의 어떤 DB 쿼리**에서 소요된 것일까?" 이 질문에 답하는 것이 바로 관찰 가능성의 마지막 기둥, \*\*'분산 추적'\*\*입니다.