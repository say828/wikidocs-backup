## AuthorizationPolicy: K8s 네트워크 정책을 넘어서는 L7 접근 제어

23장 01절에서 `PeerAuthentication`을 통해 우리는 \*\*"누가 누구와 통신할 수 있는가(Authentication)"\*\*를 mTLS로 해결했습니다. 즉, `order-service`만이 `product-service`임을 증명한 상대와 통신하도록 만들었습니다.

하지만 아직 해결되지 않은 질문이 있습니다. "그래서 `order-service`가 `product-service`의 \*\*'모든 API'\*\*를 호출해도 괜찮은가?" 이것이 바로 **인가(Authorization)**, 즉 "무엇을 할 수 있는가"의 영역입니다.

쿠버네티스에도 `NetworkPolicy`라는 기본 방화벽 기능이 있지만, 이는 L3/L4(IP, Port) 레벨에서 동작합니다. "A Pod는 B Pod와 통신할 수 있다"는 수준의 제어만 가능할 뿐, "A Pod가 B Pod의 `/admin` 경로는 호출하면 안 된다"와 같은 애플리케이션 레벨(L7)의 세밀한 제어는 불가능합니다.

-----

### Istio AuthorizationPolicy: "우리 집 현관문의 스마트 도어락" 🚪

\*\*`AuthorizationPolicy`\*\*는 Istio 서비스 메시 내에서 워크로드에 대한 접근 제어를 L7 레벨까지 확장하는 강력한 리소스입니다.

  * **쿠버네티스 `NetworkPolicy`:** 우리 집 \*\*'대문'\*\*과 같습니다. "우리 가족(특정 레이블을 가진 Pod)만 들어올 수 있다"는 규칙을 설정합니다.
  * **Istio `AuthorizationPolicy`:** 각 방마다 설치된 \*\*'스마트 도어락'\*\*과 같습니다. "아빠(특정 서비스 계정)만 서재(`GET /admin/stats`)에 들어갈 수 있고, 아이들(다른 서비스)은 거실(`GET /products`)에만 들어갈 수 있다"와 같이 훨씬 더 세분화된 규칙을 설정할 수 있습니다.

`AuthorizationPolicy`는 다음과 같은 요소들을 조합하여 규칙을 만듭니다.

1.  **`selector`:** 이 정책이 적용될 **'대상'** 워크로드(Pod)를 선택합니다.
2.  **`action`:** 정책에 일치하는 요청을 어떻게 처리할지 결정합니다. (`ALLOW`: 허용, `DENY`: 거부)
3.  **`rules`:** 어떤 요청을 허용하거나 거부할지에 대한 **'조건'** 목록입니다.
      * **`from`:** 누가 요청을 보냈는가? (Source)
          * `principals`: mTLS 인증서의 신원(Service Account)
      * **`to`:** 어떤 작업을 요청했는가? (Operation)
          * `methods`: HTTP 메서드 (`GET`, `POST`)
          * `paths`: HTTP 경로 (`/api/v1/products`)
          * `ports`: 포트 번호

-----

### 실전: `product-service`의 중요 API 보호하기

**시나리오:** `product-service`에는 두 가지 종류의 API가 있다고 가정해 봅시다.

  * \*\*`GET /api/v1/products/**`: 상품 정보를 조회하는 API. 모든 서비스가 호출할 수 있어야 합니다.
  * **`POST /api/v1/products/internal/decrease-stock`**: 재고를 직접 차감하는 매우 민감한 내부용 API. **오직 `order-service`만**이 호출할 수 있어야 합니다.

이 규칙을 `AuthorizationPolicy`로 구현해 봅시다.

#### 1단계 (사전 작업): 기본 Deny All 정책 적용

제로 트러스트 원칙에 따라, 먼저 `product-service`로 가는 \*\*모든 트래픽을 기본적으로 거부(Deny All)\*\*하는 정책을 적용합니다.

```yaml
# product-service-deny-all.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-deny-all
spec:
  # 1. product-service에 이 정책을 적용
  selector:
    matchLabels:
      app: product-service
  # 2. 아무런 rules가 없으면 모든 요청에 매치됨
  #    action이 명시되지 않으면 기본값은 ALLOW.
  #    따라서 빈 rules를 가진 ALLOW 정책은 "모두 허용"
  #    규칙이 없는 DENY 정책은 존재하지 않으므로, ALLOW 정책을 명시적으로 만듦.
  #    (실제로는 기본 ALLOW 후 특정 경로 DENY 또는 기본 DENY 후 특정 경로 ALLOW)
  #    여기서는 명시적 ALLOW 정책을 만들어 제어.
  action: ALLOW
  # rules: [] # 규칙이 없으면 모든 요청 허용
```

> **수정:** 제로 트러스트를 위해, 우리는 먼저 `product-service`로 향하는 **모든** 요청을 허용하되, 특정 조건에 맞는 요청만 **선별적으로** 허용하는 `ALLOW` 정책을 여러 개 둘 것입니다. 만약 아무런 `ALLOW` 정책과도 매치되지 않으면, Istio의 기본 동작은 \*\*암묵적 거부(Implicit Deny)\*\*입니다.

#### 2단계: 선별적 `ALLOW` 정책 작성

이제 필요한 트래픽만 명시적으로 허용하는 정책들을 추가합니다.

```yaml
# product-service-allow-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-allow-policy
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
    # --- 규칙 1: 상품 조회 API는 모두에게 허용 ---
    - to:
        - operation:
            methods: ["GET"]
            paths: ["/api/v1/products*", "/actuator/health*"] # 헬스 체크 경로도 필수
    
    # --- 규칙 2: 재고 차감 API는 오직 order-service에게만 허용 ---
    - from:
        - source:
            # 1. 요청을 보낸 주체의 Service Account가 'order-service-sa'일 때
            principals: ["cluster.local/ns/default/sa/order-service-sa"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/v1/products/internal/decrease-stock"]
```

  * **`principals`**: mTLS 핸드셰이크를 통해 확인된 클라이언트의 신원입니다. `order-service`의 `Deployment`에 `serviceAccountName: order-service-sa`를 지정해야 합니다.

#### 3단계: 정책 적용 및 테스트

```bash
kubectl apply -f product-service-allow-policy.yaml
```

이제 정책이 적용되었습니다.

  * `member-service` Pod에서 `product-service`로 `GET /api/v1/products/1`을 호출하면? **성공 (200 OK)**
  * `member-service` Pod에서 `product-service`로 `POST /api/v1/products/internal/decrease-stock`를 호출하면? **실패 (403 Forbidden)**
  * `order-service` Pod에서 `product-service`로 `POST /api/v1/products/internal/decrease-stock`를 호출하면? **성공 (200 OK)**

-----

**결론적으로,** Istio `AuthorizationPolicy`는 쿠버네티스의 L3/L4 `NetworkPolicy`를 훨씬 뛰어넘는, **애플리케이션 레벨(L7)의 세분화된 접근 제어**를 가능하게 합니다. mTLS로 \*\*"누구인지(Authentication)"\*\*를 증명하고, `AuthorizationPolicy`로 \*\*"무엇을 할 수 있는지(Authorization)"\*\*를 제어함으로써, 우리는 코드 변경 없이도 매우 강력하고 유연한 제로 트러스트 보안 모델을 서비스 메시에 구현할 수 있습니다.