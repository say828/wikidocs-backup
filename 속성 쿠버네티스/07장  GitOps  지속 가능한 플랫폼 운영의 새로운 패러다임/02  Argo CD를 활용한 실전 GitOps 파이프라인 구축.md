## 02\. Argo CD를 활용한 실전 GitOps 파이프라인 구축

GitOps의 원칙을 이해했다면, 이제 그 원칙을 현실에서 구현할 충실한 에이전트를 선택할 시간이다. CNCF의 졸업 프로젝트(Graduated Project)이자, 오늘날 GitOps 생태계의 사실상 표준으로 자리 잡은 도구가 바로 **Argo CD**다.

Argo CD는 쿠버네티스 클러스터 위에서 동작하는 컨트롤러로, Git 저장소를 감시하며 클러스터의 상태를 Git에 선언된 상태와 동기화하는 모든 복잡한 작업을 대신 처리해준다. 그 아키텍처는 GitOps 철학을 그대로 반영한다.

Argo CD의 핵심 컴포넌트는 다음과 같다.

  * **API Server:** 웹 UI나 CLI(`argocd`)가 사용하는 gRPC/REST API를 제공한다.
  * **Repository Server:** Git 저장소에 연결하여 매니페스트를 가져오고 캐싱하는 역할을 담당한다.
  * **Application Controller:** Argo CD의 심장이다. `Application`이라는 CRD를 감시하며, 실제 동기화 작업을 오케스트레이션한다. Git의 상태와 클러스터의 라이브 상태를 비교하고, 차이가 발생하면 동기화(Sync) 상태를 `OutOfSync`로 변경하며, 자동 동기화가 설정된 경우 클러스터를 원하는 상태로 조정한다.

Argo CD를 이용한 GitOps 파이프라인 구축은 크게 세 단계로 이루어진다.

**1단계: Git 저장소 준비**
먼저, 우리는 애플리케이션의 쿠버네티스 매니페스트를 담을 Git 저장소를 준비해야 한다. 이 저장소는 우리의 '이상적인 상태'를 담는 진실의 원천이 된다. 커스터마이즈나 헬름을 사용한다면, 해당 구조에 맞게 파일을 구성한다.

```
my-app-repo/
└── my-app/
    ├── kustomization.yaml
    ├── deployment.yaml
    └── service.yaml
```

**2단계: Argo CD에 `Application` 등록**
다음으로, 우리는 Argo CD에게 "이 Git 저장소의 이 경로를 저 클러스터의 저 네임스페이스와 동기화해줘"라고 알려줘야 한다. 이 '지시서' 역할을 하는 것이 바로 `Application`이라는 이름의 커스텀 리소스(CRD)다.

이 `Application` CRD는 일반적으로 Argo CD가 설치된 클러스터에 `kubectl apply`를 통해 등록한다. (닭이 먼저냐 달걀이 먼저냐 하는 문제처럼 보이지만, 보통 Argo CD 자체와 `Application` CRD들은 플랫폼 관리자가 별도의 관리 파이프라인을 통해 관리한다.)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source: # 1. "어디에서(Source)?"
    repoURL: 'https://github.com/my-org/my-app-repo.git' # Git 저장소 주소
    targetRevision: HEAD # 추적할 브랜치/태그/커밋
    path: my-app # 저장소 내의 경로

  destination: # 2. "어디로(Destination)?"
    server: 'https://kubernetes.default.svc' # 대상 클러스터 주소 (자기 자신)
    namespace: my-app-prod # 배포할 네임스페이스

  syncPolicy: # 3. "어떻게(Sync Policy)?"
    automated: # 자동 동기화 정책
      prune: true # Git에 없는 리소스는 클러스터에서 자동 삭제
      selfHeal: true # 드리프트 발생 시 자동으로 Git 상태로 복원
```

**3단계: 동기화와 시각화**
이 `Application` 매니페스트가 클러스터에 적용되는 순간, Argo CD는 즉시 일을 시작한다.

  * Git 저장소에 연결하여 `my-app` 경로의 매니페스트를 가져온다.
  * `my-app-prod` 네임스페이스의 현재 상태와 비교한다.
  * `syncPolicy.automated`가 활성화되어 있으므로, 차이가 있다면 즉시 Git의 내용에 맞춰 클러스터를 변경한다.

이 모든 과정은 Argo CD의 멋진 웹 UI를 통해 실시간으로 시각화된다. 우리는 어떤 리소스가 동기화되었는지, 어떤 리소스가 `OutOfSync` 상태인지, 각 리소스 간의 관계는 어떻게 되는지를 한눈에 파악할 수 있다.

이제부터 개발자가 할 일은 오직 `my-app-repo` 저장소에 Pull Request를 보내는 것뿐이다. PR이 리뷰되고 `main` 브랜치에 병합되면, Argo CD는 몇 분 안에 자동으로 변경 사항을 감지하고 프로덕션 클러스터에 안전하게 반영할 것이다. 더 이상 `kubectl`은 필요 없다.

하지만 애플리케이션의 수가 100개가 되면, 우리는 `Application` CRD 100개를 관리해야 하는 새로운 문제에 봉착한다. 이 문제를 우아하게 해결하는 방법이 바로 'App of Apps' 패턴이다.