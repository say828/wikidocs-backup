# 22장: Istio를 활용한 트래픽 제어와 Resilience의 외부화

## 00\. Spring Cloud Gateway에서 Istio Ingress Gateway로 트래픽 진입점 전환

21장에서 우리는 Istio를 성공적으로 설치하고 모든 마이크로서비스에 Envoy 사이드카를 주입했습니다. 이제 우리 클러스터에는 두 종류의 '게이트웨이'가 존재합니다.

1.  **Spring Cloud Gateway:** 04장에서 우리가 직접 만든 '애플리케이션 레벨' 게이트웨이.
2.  **Istio Ingress Gateway:** 21장에서 Istio를 설치할 때 자동으로 생성된 '인프라 레벨' 게이트웨이.

외부 인터넷 트래픽이 우리 MSA 시스템으로 들어오는 \*\*'최초의 관문'\*\*은 이제 둘 중 누가 되어야 할까요? 정답은 **Istio Ingress Gateway**입니다.

-----

### Istio Ingress Gateway: 서비스 메시의 공식 현관문

**Istio Ingress Gateway**는 특별한 컴포넌트가 아닙니다. 21장에서 배운 **Envoy 프록시**와 동일하지만, 서비스 메시의 '가장자리(Edge)'에 위치하여 외부 세계로부터 들어오는 모든 트래픽(North-South 트래픽)을 처리하도록 특별히 설정된 Envoy 프록시입니다.

우리가 `application.yml`로 제어했던 Spring Cloud Gateway와 달리, Istio Ingress Gateway는 Istio의 컨트롤 플레인(`Istiod`)과 다음과 같은 쿠버네티스 CRD(Custom Resource Definition)를 통해 제어됩니다.

  * **`Gateway` 리소스:** Ingress Gateway 프록시의 특정 포트를 열고, 해당 포트가 어떤 호스트 이름(예: `api.ecommerce.com`)과 프로토콜(HTTP, HTTPS)을 처리할지 정의합니다. (비유: "우리 빌딩 정문을 `api.ecommerce.com` 손님을 위해 80번 포트로 개방한다.")
  * **`VirtualService` 리소스:** `Gateway` 리소스를 통해 들어온 트래픽을 클러스터 내부의 어떤 서비스로 보낼지 **라우팅 규칙**을 정의합니다. (비유: "정문으로 들어온 `api.ecommerce.com` 손님은 1층의 `front-desk` 서비스로 안내하라.")

-----

### 왜 Istio Ingress Gateway로 전환해야 하는가?

1.  **통합된 제어 평면 (Unified Control Plane):**
    외부에서 들어오는 트래픽(North-South)과 서비스 간 내부 통신(East-West) 모두를 **`VirtualService`라는 동일한 리소스**를 사용하여 일관된 방식으로 제어할 수 있습니다. 트래픽 관리의 모든 책임이 Istio라는 단일 제어 평면으로 통합됩니다.

2.  **강력한 Edge 기능:**
    Ingress Gateway는 완전한 기능을 갖춘 Envoy 프록시이므로, TLS 종료(Termination), 복잡한 L7 라우팅, 재시도, 타임아웃, 웹 방화벽(WAF) 통합 등 강력한 Edge 기능을 애플리케이션 코드와 무관하게 인프라 레벨에서 처리할 수 있습니다.

3.  **Spring Cloud Gateway의 역할 재정의:**
    그렇다고 우리가 만든 Spring Cloud Gateway가 쓸모없어지는 것은 아닙니다. Istio Ingress Gateway가 1차 관문 역할을 하고, 그 뒤에서 Spring Cloud Gateway가 **'API Façade'** 또는 **'BFF(Backend for Frontend)'** 게이트웨이로서 더 높은 수준의 애플리케이션 특화 로직을 수행하도록 역할을 재정의할 수 있습니다.

      * **요청/응답 변환:** 여러 마이크로서비스의 응답을 조합(Aggregation)하여 모바일 클라이언트에 최적화된 단일 응답으로 만들어주는 역할.
      * **프로토콜 변환:** 외부의 REST 요청을 내부의 gRPC 서비스로 변환해주는 역할.

**새로운 트래픽 흐름:**

> **외부 인터넷 → Istio Ingress Gateway (L7 라우팅/보안) → (선택적) Spring Cloud Gateway (API 조합/변환) → 내부 마이크로서비스**

-----

### 실전: Istio Ingress Gateway 설정하기

이제 외부 트래픽이 Istio Ingress Gateway를 거쳐, 04장에서 만들었던 `gateway-service`(구 Spring Cloud Gateway)로 전달되도록 설정해 봅시다.

#### 1\. `Gateway` 리소스 작성

`istio-ingressgateway`가 80번 포트에서 `*`(모든 호스트)의 HTTP 트래픽을 받도록 설정합니다.

```yaml
# gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: ecommerce-gateway
spec:
  selector:
    istio: ingressgateway # 1. Istio의 기본 Ingress Gateway 컨트롤러를 사용
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*" # 2. 모든 호스트 이름에 대한 요청을 받음
```

#### 2\. `VirtualService` 리소스 작성

위 `Gateway`로 들어온 모든 트래픽을 우리의 `gateway-service` 쿠버네티스 서비스로 라우팅합니다.

```yaml
# virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ecommerce-virtualservice
spec:
  hosts:
    - "*" # 1. 이 VirtualService가 적용될 호스트 (Gateway와 일치)
  gateways:
    - ecommerce-gateway # 2. 이 VirtualService를 위에서 만든 Gateway에 바인딩
  http:
    - route:
        - destination:
            # 3. 트래픽을 보낼 목적지 쿠버네티스 서비스
            host: gateway-service 
            port:
              number: 8080 # gateway-service가 리스닝하는 포트
```

#### 적용 및 확인

```bash
kubectl apply -f gateway.yaml
kubectl apply -f virtualservice.yaml

# Ingress Gateway의 외부 IP 주소 확인
kubectl get svc istio-ingressgateway -n istio-system
# EXTERNAL-IP         ...
# 34.64.123.123       ...

# Ingress Gateway의 외부 IP로 curl 요청
curl http://34.64.123.123/api/v1/members/1
```

이제 요청은 Istio Ingress Gateway를 통해 우리의 `gateway-service`로 성공적으로 라우팅됩니다. 우리는 트래픽 관리의 제어권을 애플리케이션에서 인프라로 성공적으로 이전했습니다.

이제 이 강력한 Istio의 라우팅 기능을 활용하여, 다음 절부터 카나리 배포와 같은 고급 트래픽 제어 패턴을 구현해 보겠습니다.