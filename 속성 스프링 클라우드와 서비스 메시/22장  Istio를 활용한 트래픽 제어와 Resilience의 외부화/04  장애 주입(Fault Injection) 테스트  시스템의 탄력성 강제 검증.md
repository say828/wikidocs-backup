## 장애 주입(Fault Injection) 테스트: 시스템의 탄력성 강제 검증

22장 03절에서 우리는 Istio를 이용해 재시도(Retry)와 서킷 브레이커(Circuit Breaker)라는 강력한 안정성 장치를 설정했습니다. 하지만 여기서 최고의 실무자는 다음과 같은 질문을 던져야 합니다.

> **"우리가 설정한 서킷 브레이커가 실제 장애 상황에서 정말 의도한 대로 동작할 것이라고 어떻게 확신할 수 있는가?"**

실제 프로덕션 환경에서 `product-service`가 다운되기만을 기다렸다가 테스트해볼 수는 없습니다. 우리는 **선제적으로, 그리고 통제된 방식**으로 시스템의 실패 시나리오를 테스트하여, 우리가 구축한 안정성 매커니즘이 제대로 작동하는지 \*\*'증명'\*\*해야 합니다.

이것이 바로 \*\*장애 주입(Fault Injection)\*\*의 목적입니다.

-----

### Istio Fault Injection: 우리 MSA를 위한 '소방 훈련' 🔥

**장애 주입**은 **애플리케이션 코드를 단 한 줄도 변경하지 않고**, 서비스 메시의 트래픽 흐름에 의도적으로 \*\*'지연(Delay)'\*\*이나 \*\*'중단(Abort)'\*\*과 같은 결함을 주입하는 기술입니다. 이는 우리 시스템의 안정성을 검증하기 위한 '소방 훈련'과 같습니다. 실제 불이 나기를 기다리는 대신, 훈련을 통해 스프링클러와 화재 경보기가 제대로 작동하는지 미리 확인하는 것입니다.

Istio는 `VirtualService` 리소스를 통해 두 가지 유형의 장애를 매우 쉽게 주입할 수 있습니다.

1.  **지연 (Delay):**

      * 요청에 의도적으로 지연 시간을 추가합니다.
      * **목적:** 다운스트림 서비스의 응답이 느려졌을 때, 업스트림 서비스의 **타임아웃(Timeout)** 설정이 올바르게 동작하는지 검증하기 위해 사용합니다.
      * **예시:** "`product-service`로 가는 요청의 10%에 대해, 5초의 고정 지연 시간을 추가하라."

2.  **중단 (Abort):**

      * 요청을 의도적으로 중단시키고, 지정된 HTTP 에러 코드를 반환합니다.
      * **목적:** 다운스트림 서비스가 실패했을 때, 업스트림 서비스의 \*\*재시도(Retry)\*\*나 **서킷 브레이커** 정책이 올바르게 동작하는지 검증하기 위해 사용합니다.
      * **예시:** "`product-service`로 가는 요청의 20%에 대해, 즉시 HTTP `503 Service Unavailable` 에러로 응답하라."

-----

### 실전 테스트: 서킷 브레이커 강제 개방(Open)시키기

**시나리오:** 22장 03절에서 설정한 `product-service`의 서킷 브레이커("5번 연속 5xx 에러 발생 시 1분간 차단")가 실제로 동작하는지 검증해 봅시다.

#### 1\. 장애 주입을 위한 `VirtualService` 작성

`product-service`로 가는 **모든(100%) 트래픽**을 강제로 HTTP `503` 에러로 중단시키는 `VirtualService` 규칙을 작성합니다.

```yaml
# fault-injection-vs.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-vs # 기존 VirtualService와 이름이 같아야 덮어써짐
spec:
  hosts:
    - product-service
  http:
    - route:
        - destination:
            host: product-service
            subset: v1
      # --- 장애 주입 규칙 ---
      fault:
        abort:
          # 1. 100%의 요청에 대해
          percentage:
            value: 100.0
          # 2. HTTP 503 에러를 반환
          httpStatus: 503
```

#### 2\. 장애 주입 시작

이 YAML 파일을 클러스터에 적용하는 순간, '소방 훈련'이 시작됩니다.

```bash
kubectl apply -f fault-injection-vs.yaml
```

이제 `order-service`가 `product-service`를 호출하는 모든 시도는 즉시 `503` 에러를 응답받게 됩니다.

#### 3\. 시스템 동작 관찰

  * **Grafana 대시보드:** `product-service`의 5xx 에러율 그래프가 100%로 치솟는 것을 실시간으로 확인합니다.
  * **서킷 브레이커 동작:** 5번의 요청이 연속으로 실패하면, `DestinationRule`에 정의된 \*\*`outlierDetection`(서킷 브레이커)\*\*이 발동합니다.
  * **Istio 상태 확인:** `istioctl` CLI를 사용하여 `order-service`의 사이드카 프록시가 `product-service`의 Pod를 실제로 로드 밸런싱 풀에서 제외했는지 확인할 수 있습니다.
    ```bash
    # order-service의 Pod 이름으로 조회
    istioctl proxy-config cluster <order-service-pod-name> --fqdn product-service...
    # 출력에서 특정 Pod의 상태가 'HEALTHY'에서 'UNHEALTHY(Ejected)'로 변경된 것을 확인
    ```

#### 4\. 장애 주입 종료

테스트가 끝나면, 장애 주입 규칙을 제거하여 시스템을 정상 상태로 되돌립니다.

```bash
kubectl delete -f fault-injection-vs.yaml

# 만약 이전에 v1/v2 라우팅 규칙이 있었다면, 해당 VirtualService를 다시 apply
kubectl apply -f product-service-vs.yaml 
```

장애 상황이 해소되면, 서킷 브레이커는 `baseEjectionTime`(1분)이 지난 후 `HALF_OPEN` 상태를 거쳐 다시 `CLOSED` 상태로 자동으로 복구될 것입니다.

-----

장애 주입은 \*\*'카오스 엔지니어링(Chaos Engineering)'\*\*의 핵심적인 실천 방법입니다. 이를 통해 우리는 우리 시스템이 '아마도 안정적일 것이다'라고 **희망**하는 수준을 넘어, '분명히 안정적이다'라고 **증명**할 수 있게 됩니다. Istio는 이 복잡한 테스트 기법을 간단한 YAML 선언만으로 가능하게 하여, 우리 시스템의 탄력성에 대한 강력한 자신감을 심어줍니다.