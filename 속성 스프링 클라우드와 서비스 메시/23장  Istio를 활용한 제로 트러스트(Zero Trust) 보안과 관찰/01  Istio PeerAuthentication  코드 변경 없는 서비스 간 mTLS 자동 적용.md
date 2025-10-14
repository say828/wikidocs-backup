## Istio PeerAuthentication: 코드 변경 없는 서비스 간 mTLS 자동 적용

23장 00절에서 우리는 mTLS가 MSA 보안의 핵심임을 배웠고, Istio가 이를 자동으로 처리해 준다고 했습니다. 이제 "어떻게?"에 대한 답을 할 차례입니다. Istio는 \*\*`PeerAuthentication`\*\*이라는 매우 단순하고 선언적인 리소스를 통해 메시 전체의 mTLS 정책을 제어합니다.

-----

### `PeerAuthentication` 리소스: mTLS 모드 설정하기

`PeerAuthentication` 리소스는 특정 워크로드(서비스)가 다른 워크로드로부터 들어오는 트래픽을 어떻게 처리할지를 정의합니다. 핵심 설정은 `mtls.mode`이며, 세 가지 모드 중 하나를 선택할 수 있습니다.

1.  **`PERMISSIVE` (관용 모드) - 기본값:**

      * "나는 **mTLS 암호화 트래픽**과 **일반 텍스트(Plaintext) 트래픽**을 모두 받겠다."
      * Istio를 처음 설치했을 때의 기본 모드입니다. 이는 서비스 메시 외부(사이드카가 없는)의 레거시 서비스와 메시 내부의 서비스가 점진적으로 공존할 수 있도록 해주는, 마이그레이션을 위한 모드입니다. 메시 내의 클라이언트가 메시 내의 서버를 호출하면, 통신은 **자동으로 mTLS로 업그레이드**됩니다.

2.  **`STRICT` (엄격 모드):**

      * "나는 **오직 mTLS 암호화 트래픽**만 받겠다. 일반 텍스트 트래픽은 모두 거부한다."
      * 이것이 바로 **제로 트러스트(Zero Trust) 네트워크**를 완성하는 최종 목표입니다. 이 모드가 적용되면, 사이드카가 없는 클라이언트나 mTLS 핸드셰이크에 실패한 클라이언트는 해당 서비스에 아예 접근할 수 없습니다.

3.  **`DISABLE` (비활성화 모드):**

      * "나는 mTLS를 사용하지 않고, **오직 일반 텍스트 트래픽**만 받겠다."
      * 특별한 이유가 없는 한 프로덕션 환경에서는 절대 사용해서는 안 되는 모드입니다.

이 정책은 **메시 전체, 네임스페이스 단위, 또는 특정 워크로드 단위**로 적용하여 매우 유연하게 보안 수준을 제어할 수 있습니다.

-----

### 실전: 메시 전체에 `STRICT` mTLS 강제하기

이제 단 하나의 YAML 파일을 이용하여, 우리 이커머스 MSA 전체의 내부 통신을 `STRICT` mTLS 모드로 전환해 보겠습니다.

#### 1단계: 현재 mTLS 상태 확인

`istioctl` CLI는 현재 서비스 간의 mTLS 상태를 진단하는 강력한 도구를 제공합니다.

```bash
# istioctl authn tls-check <source-pod> <destination-service>
# 예: gateway-service Pod에서 product-service로의 통신 상태 확인
istioctl authn tls-check gateway-service-pod-xyz.default product-service.default.svc.cluster.local

# 출력 예시 (PERMISSIVE 모드일 때)
# HOST:PORT                                STATUS     SERVER        CLIENT     CLIENT.POD
# product-service.default.svc.cluster.local:8082   OK         PERMISSIVE    MTLS       gateway-service-pod-xyz.default
```

  * `SERVER: PERMISSIVE`: 목적지(`product-service`)가 `PERMISSIVE` 모드임을 의미합니다.
  * `CLIENT: MTLS`: 클라이언트(`gateway-service`)가 mTLS를 사용하여 통신하고 있음을 의미합니다.

#### 2단계: 메시 전체에 적용되는 `PeerAuthentication` 정책 작성

메시 전체에 정책을 적용하려면, Istio의 컨트롤 플레인이 설치된 **`istio-system` 네임스페이스**에 리소스를 생성합니다.

```yaml
# strict-mtls-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default-mesh-wide-mtls"
  # 1. 컨트롤 플레인의 네임스페이스에 정책을 생성하면 메시 전체에 적용
  namespace: "istio-system" 
spec:
  mtls:
    # 2. 모든 서비스 간 통신에 STRICT 모드를 강제
    mode: STRICT
```

#### 3단계: 정책 적용 및 확인

```bash
kubectl apply -f strict-mtls-policy.yaml
```

이제 정책이 적용될 때까지 잠시 기다린 후, 다시 mTLS 상태를 확인합니다.

```bash
istioctl authn tls-check gateway-service-pod-xyz.default product-service.default.svc.cluster.local

# 출력 예시 (STRICT 모드로 변경됨)
# HOST:PORT                                STATUS     SERVER     CLIENT     CLIENT.POD
# product-service.default.svc.cluster.local:8082   OK         STRICT     MTLS       gateway-service-pod-xyz.default
```

`SERVER` 상태가 `STRICT`로 변경된 것을 확인할 수 있습니다.

이제 클러스터 내부에 사이드카가 없는 테스트용 Pod를 하나 띄우고, 그 안에서 `product-service`로 `curl` 호출을 시도하면 어떻게 될까요?
`curl http://product-service:8082/` -\> **Connection reset by peer**
일반 텍스트(Plaintext) 통신이 `product-service`의 Envoy 프록시 레벨에서 차단되었음을 의미합니다\!

-----

**결론적으로,** 우리는 **애플리케이션 코드를 전혀 변경하지 않고**, 단 하나의 간단한 `YAML` 파일을 클러스터에 적용하는 것만으로 우리 MSA 시스템의 모든 내부 통신을 **완벽하게 암호화하고 상호 인증**하도록 만들었습니다.

이는 서비스 메시가 제공하는 가장 강력한 가치 중 하나이며, 복잡한 보안 요구사항을 개발자의 부담에서 플랫폼의 책임으로 이전하는 '플랫폼 엔지니어링'의 핵심 사상을 보여줍니다. 이제 이 강력한 인증 계층 위에, "누가 무엇을 할 수 있는가"를 제어하는 '인가' 정책을 다음 절에서 구현해 보겠습니다.