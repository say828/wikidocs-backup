## 카나리(Canary) 배포: VirtualService를 이용한 정교한 트래픽 시프팅(Traffic Shifting)

20장에서 우리는 카나리 배포를 "새로운 버전을 아주 소수의 사용자에게만 먼저 공개하여 위험을 최소화하는" 가장 진보된 배포 전략이라고 소개했습니다. 하지만 당시에는 이 정교한 트래픽 분배를 어떻게 구현할지에 대한 구체적인 방법이 없었습니다.

이제 우리는 Istio의 \*\*`VirtualService`\*\*라는 강력한 무기를 손에 쥐었습니다. `VirtualService`의 **가중치 기반 라우팅(Weight-based Routing)** 기능을 사용하면, 코드 한 줄 변경 없이 YAML 파일 수정만으로 실제 운영 트래픽을 백분율(%) 단위로 정밀하게 제어할 수 있습니다.

-----

### 카나리 배포 시나리오

**목표:** 새로 개발된 `product-service:v2` 버전을 전체 사용자 중 \*\*10%\*\*에게만 먼저 노출하고, 안정성을 검증한 뒤 100%로 확대하고자 합니다.

22장 02절에서 우리는 `DestinationRule`을 통해 `v1`과 `v2`라는 두 개의 서비스 서브셋을 이미 정의해 두었습니다. 이제 남은 일은 이 두 서브셋으로 트래픽을 어떻게 나눌지 `VirtualService`에 지시하는 것뿐입니다.

#### `VirtualService` 수정: 트래픽 가중치 부여

기존에 모든 트래픽을 `v1`으로 보냈던 `product-service-vs.yaml` 파일을 다음과 같이 수정합니다.

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
        # 1. 이제 트래픽을 보낼 목적지가 두 개 이상
        - destination:
            host: product-service
            subset: v1
          # 2. v1 서브셋으로는 전체 트래픽의 90%를 보낸다
          weight: 90
        - destination:
            host: product-service
            subset: v2
          # 3. v2 서브셋으로는 전체 트래픽의 10%를 보낸다
          weight: 10
```

  * **`weight` 필드:** `route` 블록 내의 각 `destination`에 대한 트래픽 가중치를 백분율로 지정합니다. Istio는 이 가중치에 따라 Envoy 프록시가 트래픽을 분배하도록 동적으로 설정합니다. **같은 `route` 블록 내 모든 `weight`의 합은 반드시 100이어야 합니다.**

이 수정된 YAML 파일을 `kubectl apply -f product-service-vs.yaml` 명령어로 클러스터에 적용하는 순간, Istio 컨트롤 플레인(`Istiod`)은 이 새로운 규칙을 모든 사이드카 프록시에 전파하고, **즉시** `product-service`로 향하는 전체 트래픽의 약 10%가 `v2` Pod로 흐르기 시작합니다.

-----

### 완전한 카나리 릴리즈 프로세스

1.  **초기 배포 (10%):** 위 YAML을 적용하여 10%의 트래픽을 `v2`로 보냅니다.
2.  **모니터링 및 검증:** 18장에서 구축한 Grafana 대시보드를 주시합니다. `product-service:v2` Pod들만의 에러율(5xx), 응답 시간(99th percentile latency), CPU/메모리 사용량을 **`v1` Pod들과 비교 분석**합니다.
      * Prometheus의 다차원 레이블 모델 덕분에 `sum(rate(http_requests_total{app="product-service", version="v2"}[5m]))` 과 같이 버전별로 메트릭을 정밀하게 분리하여 볼 수 있습니다.
3.  **점진적 확대 (Progressive Delivery):** `v2`가 안정적이라고 판단되면, `VirtualService`의 `weight` 값을 점진적으로 변경하여 `v2`로 가는 트래픽을 늘려나갑니다.
      * `weight: v1(70) / v2(30)` → `weight: v1(50) / v2(50)` → `weight: v1(0) / v2(100)`
4.  **롤아웃 완료:** 모든 트래픽이 `v2`로 성공적으로 전환되면, 더 이상 필요 없는 `v1` Deployment를 클러스터에서 제거합니다.
5.  **신속한 롤백:** 만약 10% 트래픽 단계에서 `v2`의 에러율이 급증하는 것이 감지되면, 즉시 `VirtualService`의 `weight`를 `v1(100) / v2(0)`으로 되돌려 장애의 영향을 최소화하고 원인 분석에 들어갑니다.

**결론적으로,** `VirtualService`를 이용한 트래픽 시프팅은 카나리 배포를 위한 가장 강력하고 유연한 방법입니다. 이를 통해 우리는 실제 운영 환경의 실제 사용자 트래픽을 대상으로, 새로운 버전에 대한 **데이터 기반의 자신감**을 가지고 배포할 수 있게 됩니다. 이는 "배포 버튼을 누르는 것에 대한 두려움"을 없애고, 빠르고 안전한 릴리즈 사이클을 가능하게 하는 MSA 운영의 핵심 기술입니다.