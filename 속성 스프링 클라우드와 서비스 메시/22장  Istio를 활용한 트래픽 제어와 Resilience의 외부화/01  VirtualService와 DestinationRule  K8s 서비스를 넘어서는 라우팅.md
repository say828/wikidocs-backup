## VirtualService와 DestinationRule: K8s 서비스를 넘어서는 라우팅

22장 00절에서 우리는 `Gateway`와 `VirtualService`를 이용해 외부 트래픽을 우리 서비스 메시 내부로 들여왔습니다. 이제 서비스 메시의 트래픽 제어 능력을 본격적으로 탐구할 시간입니다.

쿠버네티스의 기본 `Service` 오브젝트는 훌륭하지만, L4(TCP) 레벨에서 동작하는 단순한 로드 밸런서에 가깝습니다. 즉, `Service`는 자신에게 온 트래픽을 자신이 알고 있는 Pod들에게 **아무런 조건 없이 무작위로 분배**할 뿐입니다.

하지만 실제 운영 환경에서는 훨씬 더 정교한 제어가 필요합니다.

  * "/v2/products" 경로로 오는 요청은 **새로운 `v2` 버전의 Pod**로 보내고 싶다.
  * "User-Agent" 헤더에 "iphone"이 포함된 요청은 **모바일 전용 `v-mobile` 버전의 Pod**로 보내고 싶다.
  * 전체 트래픽의 90%는 안정적인 `v1` 버전으로, 10%는 테스트를 위해 `v2` 버전으로 나누어 보내고 싶다.

쿠버네티스 `Service`는 이러한 L7(애플리케이션) 레벨의 라우팅을 전혀 이해하지 못합니다. 이것이 바로 Istio의 \*\*`VirtualService`\*\*와 \*\*`DestinationRule`\*\*이 필요한 이유입니다.

-----

### 1\. VirtualService: "어디로 갈 것인가?" (트래픽 라우팅 규칙)

\*\*`VirtualService`\*\*는 서비스 메시 내에서 특정 서비스로 향하는 트래픽의 \*\*'라우팅 규칙'\*\*을 정의하는 핵심 리소스입니다.

  * **비유:** 사거리의 **'스마트 교통 경찰'** 👮.
    쿠버네티스 `Service`는 단지 '사거리의 주소'일 뿐입니다. `VirtualService`는 이 사거리에 서서, 들어오는 모든 차(요청)를 하나하나 살펴보고, 차종(헤더), 목적지(경로), 탑승 인원(파라미터) 등을 확인한 뒤, 어떤 길(서비스 버전)로 보낼지 결정하는 교통 경찰의 역할을 합니다.

`VirtualService`는 `match` 블록을 통해 특정 조건을 정의하고, `route` 블록을 통해 해당 조건에 맞는 트래픽을 어디로 보낼지 지정합니다.

-----

### 2\. DestinationRule: "그곳은 어떤 곳인가?" (트래픽 정책 정의)

\*\*`DestinationRule`\*\*은 `VirtualService`의 라우팅 규칙에 따라 트래픽이 **목적지에 도착한 후, 해당 트래픽에 적용될 정책**을 정의합니다.

  * **비유:** 교통 경찰(`VirtualService`)이 "저쪽 길로 가세요"라고 안내했을 때, 그 \*\*'길 자체의 규칙'\*\*을 정의하는 교통 표지판입니다. "저 길은 왕복 2차선이고(커넥션 풀 설정), 짝수 번지 집들은 'v1' 동네이며, 홀수 번지 집들은 'v2' 동네입니다(서비스 서브셋 정의)."

`DestinationRule`의 가장 중요한 역할은, 동일한 서비스(예: `product-service`)에 속한 Pod들을 특정 \*\*레이블(Label)\*\*을 기준으로 \*\*'서브셋(Subsets)'\*\*이라는 논리적인 그룹으로 나누는 것입니다.

-----

### VirtualService와 DestinationRule의 협업

두 리소스는 반드시 함께 동작합니다.

> **`DestinationRule`이 서비스의 '서브셋'을 먼저 정의해야, `VirtualService`가 해당 '서브셋'으로 트래픽을 보낼 수 있다.**

**시나리오:** `product-service`의 `v1`과 `v2` 두 가지 버전을 배포하고, 모든 트래픽을 우선 안정적인 `v1`으로 보내도록 설정해 봅시다.

#### 1단계: 버전별 Deployment 배포

먼저, `version: v1`과 `version: v2` 레이블을 가진 두 개의 `Deployment`를 배포합니다. 쿠버네티스 `Service`는 `app: product-service` 셀렉터를 통해 이 두 버전의 Pod를 모두 바라봅니다.

#### 2단계: `DestinationRule` 작성 (서브셋 정의)

`product-service`를 `v1`과 `v2`라는 두 개의 서브셋으로 정의합니다.

```yaml
# product-service-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service-dr
spec:
  # 1. 이 규칙이 적용될 호스트(쿠버네티스 서비스 이름)
  host: product-service
  # 2. 서비스의 버전을 나타내는 서브셋들을 정의
  subsets:
    - name: v1 # 서브셋의 이름
      labels:
        version: v1 # 이 서브셋에 속할 Pod를 선택하는 레이블
    - name: v2
      labels:
        version: v2
```

#### 3단계: `VirtualService` 작성 (라우팅 규칙 정의)

모든 트래픽을 `DestinationRule`에서 정의한 `v1` 서브셋으로 보내도록 라우팅 규칙을 설정합니다.

```yaml
# product-service-vs.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-vs
spec:
  # 1. 이 규칙이 적용될 호스트
  hosts:
    - product-service
  http:
    # 2. 모든 HTTP 요청에 대해
    - route:
        # 3. 다음 목적지로 트래픽을 보낸다
        - destination:
            host: product-service
            # 4. 그 중에서도 'v1'이라는 이름의 서브셋으로만 보낸다
            subset: v1
```

이제 `order-service`가 `http://product-service`를 호출하면, Istio는 이 `VirtualService` 규칙에 따라 트래픽을 100% `version: v1` 레이블을 가진 Pod에게만 전송합니다. `v2` Pod는 배포되어 있지만 어떠한 트래픽도 받지 않습니다.

`VirtualService`와 `DestinationRule`은 쿠버네티스 `Service`의 한계를 뛰어넘어, 애플리케이션 레벨(L7)의 정교한 트래픽 제어를 가능하게 하는 Istio의 핵심 도구입니다. 이 두 리소스를 마스터함으로써, 우리는 다음 절에서 다룰 '카나리 배포'와 같은 고급 배포 전략을 구현할 수 있는 강력한 기반을 갖추게 됩니다.