## Resilience4j의 외부화: Istio 서킷 브레이커와 재시도(Retry) 적용

06장에서 우리는 `Resilience4j` 라이브러리를 `order-service`에 직접 추가하여, `product-service` 호출에 대한 서킷 브레이커와 재시도 로직을 **애플리케이션 코드 레벨**에서 구현했습니다. 이 방식은 매우 효과적이지만, 21장에서 논의했듯이 다음과 같은 한계를 가집니다.

  * **언어 종속성:** 파이썬으로 만든 서비스에는 다른 라이브러리를 사용해야 합니다.
  * **중복 설정:** 재시도 정책이 변경되면, 해당 서비스를 호출하는 모든 클라이언트 서비스의 `application.yml`을 수정해야 합니다.
  * **비즈니스 로직과 인프라 로직의 혼재:** 애플리케이션 코드가 '네트워크 안정성'이라는 인프라 문제를 책임져야 합니다.

서비스 메시(Service Mesh)는 이러한 안정성 패턴을 \*\*애플리케이션 외부의 플랫폼 레벨로 '외부화(Externalize)'\*\*하는 것을 목표로 합니다. 이제 `order-service`의 `Resilience4j` 관련 코드를 모두 제거하고, 동일한 기능을 Istio를 통해 구현해 보겠습니다.

-----

### 1\. Istio 재시도 (Retry) 적용

**목표:** `product-service`로의 요청이 일시적인 네트워크 문제나 `503` 에러로 실패했을 때, 최대 3번까지 자동으로 재시도하도록 설정합니다.

재시도 정책은 "어떤 경로로 가는 요청에 적용할 것인가"를 정의하므로, **`VirtualService`** 리소스에서 설정합니다.

```yaml
# product-service-vs.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-vs
spec:
  hosts:
    - product-service
  http:
    - route:
        # ... (기존 destination 설정) ...
      
      # --- 재시도 정책 추가 ---
      retries:
        attempts: 3 # 1. 최대 3번 시도 (첫 시도 + 2번 재시도)
        perTryTimeout: 2s # 2. 각 시도별 타임아웃은 2초
        
        # 3. 어떤 조건에서 재시도할지 명시
        # connect-failure: TCP 연결 실패
        # refused-stream: upstream에서 RST 패킷으로 연결 거부
        # 503: 서비스 일시 사용 불가 상태 코드
        retryOn: connect-failure,refused-stream,503
```

이 YAML 파일을 `kubectl apply`하는 순간, `product-service`를 호출하는 **모든** 서비스(`order-service`, `gateway-service` 등)의 사이드카 프록시는 이 재시도 정책을 자동으로 적용받게 됩니다. 애플리케이션 코드는 단 한 줄도 변경되지 않았습니다.

-----

### 2\. Istio 서킷 브레이커 (Circuit Breaker) 적용

**목표:** `product-service`의 특정 Pod가 5번 연속으로 5xx 에러를 반환하면, 해당 Pod를 1분 동안 로드 밸런싱 풀에서 제외하여 장애를 격리합니다.

서킷 브레이커는 특정 '목적지(Destination)'의 건강 상태를 기준으로 동작하므로, **`DestinationRule`** 리소스에서 설정합니다. Istio에서는 이 기능을 \*\*이상 감지(Outlier Detection)\*\*라고 부릅니다.

```yaml
# product-service-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service-dr
spec:
  host: product-service
  
  # --- 서킷 브레이커 (이상 감지) 정책 추가 ---
  trafficPolicy:
    outlierDetection:
      # 1. 5번 연속으로 5xx 에러가 발생하면
      consecutive5xxErrors: 5
      # 2. 30초의 주기로 호스트들의 건강 상태를 스캔
      interval: 30s
      # 3. 해당 호스트(Pod)를 1분 동안 로드 밸런싱 풀에서 제외
      baseEjectionTime: 1m
      # 4. 전체 Pod 중 최대 50%까지만 격리 (전체 장애 방지)
      maxEjectionPercent: 50
      
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

이제 `product-service`의 10개 Pod 중 하나가 문제를 일으켜 5번 연속 503 에러를 반환하면, 클러스터 내의 모든 Envoy 프록시들은 "저 Pod는 현재 비정상이니, 1분간 트래픽을 보내지 말자"라고 스스로 판단하게 됩니다. 트래픽은 나머지 9개의 건강한 Pod로만 전달되고, 장애가 발생한 Pod는 스스로 복구할 시간을 벌게 됩니다.

-----

### Resilience4j vs. Istio: 책임의 이동

| 특징 | Resilience4j (애플리케이션 레벨) | Istio (플랫폼 레벨) |
| :--- | :--- | :--- |
| **범위** | 단일 JVM 애플리케이션 내부 | 서비스 메시 전체 (모든 언어) |
| **설정** | 각 서비스의 `application.yml`에 분산 | 중앙화된 Istio CRD (`YAML`) |
| **애플리케이션**| 라이브러리를 인지하고 코드를 작성 | **자신의 트래픽이 제어되는지 전혀 모름** |
| **업그레이드** | 애플리케이션 재배포 필요 | 서비스 메시 업그레이드 (운영팀) |
| **세분성** | 특정 Java 메서드 단위로 보호 가능 | 서비스 엔드포인트(호스트) 단위로 보호 |

**결론적으로,** 재시도와 서킷 브레이커 같은 안정성 패턴을 Istio 서비스 메시로 \*\*'외부화'\*\*함으로써, 우리는 다음과 같은 엄청난 이점을 얻습니다.

  * 애플리케이션 코드는 순수한 비즈니스 로직에만 집중하여 **단순해집니다.**
  * 안정성 정책이 중앙에서 **일관되게 관리**되며, 언어에 구애받지 않습니다.
  * 애플리케이션 재배포 없이, **운영 중에 동적으로** 안정성 정책을 튜닝할 수 있습니다.

이는 '똑똑한 애플리케이션'에서 \*\*'똑똑한 플랫폼'\*\*으로 책임이 이동하는, 클라우드 네이티브 아키텍처의 핵심적인 패러다임 전환입니다.