## 03\. [실전] Argo Rollouts를 활용한 점진적 배포 자동화

지금까지 우리가 논의한 블루/그린과 카나리 배포 전략은 강력하지만, 쿠버네티스 기본 오브젝트만으로 이를 수동 관리하는 것은 복잡하고 오류가 발생하기 쉽다. 트래픽 가중치를 조절하기 위해 여러 개의 디플로이먼트를 관리하고, 서비스 설정을 계속 변경하며, Prometheus 대시보드를 뚫어져라 쳐다보다가 문제가 생기면 허둥지둥 롤백 명령을 내리는 것은 결코 지속 가능한 방식이 아니다. 🤖

이 모든 과정을 하나의 선언적인 워크플로우로 자동화하기 위해 탄생한 것이 바로 **Argo Rollouts**다. Argo Rollouts는 쿠버네티스의 기본 `Deployment` 오브젝트를 대체하는, 훨씬 더 지능적이고 강력한 CRD(Custom Resource Definition)인 **`Rollout`** 리소스를 제공하는 컨트롤러다.

Argo Rollouts를 사용하면, 당신은 더 이상 `Deployment`를 사용하지 않는다. 대신 `Rollout` 리소스를 정의하여 애플리케이션을 배포한다. 이 `Rollout` 리소스 안에는 블루/그린 혹은 카나리 배포를 위한 모든 전략이 선언적으로 기술된다. Argo Rollouts 컨트롤러는 이 명세를 읽고, 트래픽을 정밀하게 제어하며, 심지어 배포의 성공 여부를 자동으로 판단하여 다음 단계로 진행하거나 롤백하는 모든 과정을 지휘하는 오케스트라의 마에스트로가 되어준다. 🎼

-----

### Argo Rollouts의 작동 방식

Argo Rollouts의 마법은 **트래픽 라우팅 통합**과 **분석 기반 자동화**라는 두 가지 핵심 기능에서 나온다.

1.  **트래픽 라우팅 통합:** Argo Rollouts는 단순히 파드의 개수를 조절하는 것을 넘어, \*\*서비스 메시(Istio, Linkerd)나 인그레스 컨트롤러(NGINX, Traefik)\*\*와 직접 통합된다. 이를 통해 "신버전으로 트래픽을 10%만 보내라"와 같은 L7 트래픽 가중치 조절을 매우 정밀하게 수행할 수 있다. 컨트롤러가 직접 Istio의 `VirtualService`나 NGINX Ingress의 어노테이션을 동적으로 수정하여 이 모든 것을 자동화한다.

2.  **분석 기반 자동화 (Analysis):** 이것이 Argo Rollouts의 진정한 힘이다. `Rollout` 명세 안에, 우리는 배포의 각 단계마다 수행할 '분석' 작업을 정의할 수 있다. 예를 들어, "카나리 버전으로 트래픽을 10% 보낸 후 5분 동안 Prometheus에 쿼리를 날려, HTTP 5xx 에러율이 1%를 넘지 않고, P99 응답 시간이 500ms 미만인지 확인하라"와 같은 규칙을 선언할 수 있다. 만약 이 분석이 성공하면 Rollout은 자동으로 다음 단계(예: 트래픽 25%로 증량)로 진행하고, 만약 실패하면 즉시 배포를 중단하고 자동으로 롤백을 수행한다. 더 이상 인간의 주관적인 판단이 개입할 여지가 없다.

-----

### 카나리 배포를 위한 Rollout 리소스 예제

다음은 Argo Rollouts를 이용한 점진적인 카나리 배포 전략을 정의한 `Rollout` 리소스의 예시다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app-rollout
spec:
  replicas: 10
  selector:
    matchLabels:
      app: my-app
  template: # <-- Deployment의 template과 동일하다.
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:2.0.0 # <-- 배포할 새로운 버전
        ports:
        - containerPort: 8080

  strategy:
    canary:
      # 서비스 메시/인그레스 컨트롤러를 이용한 트래픽 제어 설정
      canaryService: my-app-canary-svc # 카나리 트래픽을 위한 서비스
      stableService: my-app-stable-svc # 안정 버전 트래픽을 위한 서비스
      trafficRouting:
        istio:
          virtualService:
            name: my-app-vsvc # 제어할 Istio VirtualService
            routes:
            - primary
      
      steps: # 점진적 배포 단계 정의
      - setWeight: 10 # 1. 먼저 트래픽의 10%를 카나리로 보낸다.
      - pause: {} # 2. 수동 확인을 위해 무기한 일시정지 (kubectl argo rollouts promote로 진행)

      - setWeight: 25 # 3. 트래픽을 25%로 증량한다.
      - pause: { duration: 5m } # 4. 5분간 대기하며 상태를 관찰한다.

      # 5. 분석을 통해 자동 승인/롤백 결정
      - setWeight: 50
      - analysis:
          templates:
          - templateName: http-error-rate # 미리 정의된 분석 템플릿 사용
          args:
          - name: service-name
            value: my-app-canary-svc
          - name: error-threshold
            value: "1" # 에러율 1% 미만
      # 분석이 성공하면 나머지 트래픽이 모두 신버전으로 전환된다.
```

이 YAML 파일 하나에, 우리는 복잡한 카나리 릴리즈의 전체 생명주기를 완벽하게 선언했다. 개발자는 그저 이미지 버전만 `2.0.0`으로 변경하여 Git에 커밋하기만 하면 된다. 그러면 Argo Rollouts가 이 정교한 시나리오에 따라 한 치의 오차도 없이 배포를 오케스트레이션하고, 문제가 발생하면 아무도 모르게 조용히 롤백하여 시스템의 안정성을 지켜줄 것이다.

Argo Rollouts는 단순한 배포 도구를 넘어, 데브옵스 철학의 정점을 보여주는 플랫폼이다. 이는 배포 과정을 더 이상 위험하고 스트레스받는 이벤트가 아니라, 데이터에 기반하여 자신감 있게 수행할 수 있는 예측 가능하고 자동화된 프로세스로 바꾸어 놓는다.

이제 우리의 애플리케이션은 가장 안전하고 정교한 방식으로 프로덕션 환경에 배포될 준비를 마쳤다. 하지만 지금까지 우리가 다룬 애플리케이션은 대부분 '상태가 없는(stateless)' 것들이었다. 만약 데이터베이스처럼 상태를 기억해야 하는 애플리케이션이라면 어떨까? 다음 장에서는 이 까다로운 '상태 저장(stateful)' 워크로드를 쿠버네티스 위에서 길들이는 방법에 대해 탐험해 볼 것이다.