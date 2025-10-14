## 03\. App of Apps 패턴: 복잡한 애플리케이션 의존성 관리하기

단일 애플리케이션을 Argo CD로 관리하는 것은 간단하다. 하지만 현실의 마이크로서비스 환경에서는 수십, 수백 개의 애플리케이션과 애드온(Prometheus, Grafana, Istio 등)을 관리해야 한다. 이 모든 애플리케이션 각각에 대해 `Application` CRD YAML 파일을 만들어 수동으로 관리하는 것은 또 다른 종류의 'YAML 지옥'일 뿐이다.

**App of Apps 패턴**은 이 문제를 해결하는 매우 우아하고 강력한 GitOps 네이티브 접근법이다. 이 패턴의 핵심 아이디어는 간단하다. **"Argo CD의 `Application` 리소스 자체도 그냥 하나의 쿠버네티스 리소스일 뿐이니, 다른 `Application`을 관리하는 `Application`을 만들자\!"**

즉, 우리는 단 하나의 \*\*최상위 루트 앱(Root App)\*\*을 만든다. 이 루트 앱의 역할은 실제 비즈니스 애플리케이션을 배포하는 것이 아니라, 다른 모든 자식 애플리케이션(Child Apps)들의 `Application` CRD를 배포하고 관리하는 것이다.

이것이 어떻게 동작하는지 Git 저장소 구조를 통해 살펴보자.

```
my-platform-repo/
├── apps/
│   ├── app1.yaml       # <-- app1의 Application CRD
│   ├── app2.yaml       # <-- app2의 Application CRD
│   └── monitoring.yaml # <-- Prometheus, Grafana를 배포할 Application CRD
└── root-app.yaml       # <-- 위의 모든 CRD를 관리할 최상위 루트 앱
```

**`apps/app1.yaml` (자식 앱의 정의)**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app1
  namespace: argocd
spec:
  source:
    repoURL: 'https://github.com/my-org/app1-repo.git' # app1의 소스코드 저장소
    path: deploy/prod
  destination:
    namespace: app1-prod
  # ...
```

그리고 이 모든 자식 앱들을 배포하는 단 하나의 루트 앱을 정의한다.

**`root-app.yaml` (최상위 루트 앱)**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  source:
    repoURL: 'https://github.com/my-org/my-platform-repo.git' # 바로 이 저장소
    targetRevision: HEAD
    path: apps # <-- 'apps' 디렉토리를 감시하라!
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy:
    automated: {}
```

이제 우리가 할 일은 단 하나다. `root-app.yaml` 파일 하나만을 수동으로 클러스터에 `kubectl apply`하여 이 모든 연쇄 반응을 시작시키는 것이다.

1.  `root-app`이 클러스터에 생성된다.
2.  Argo CD는 `root-app`의 지시에 따라 `my-platform-repo` 저장소의 `apps` 디렉토리를 스캔한다.
3.  `apps` 디렉토리 안에 있는 `app1.yaml`, `app2.yaml`, `monitoring.yaml` 파일들을 발견한다.
4.  Argo CD는 이 YAML 파일들의 내용이 `Application` CRD임을 인지하고, 이들을 클러스터(`argocd` 네임스페이스)에 생성한다.
5.  이제 클러스터에는 `app1`, `app2`, `monitoring`이라는 세 개의 새로운 `Application` 리소스가 생겨났다. Argo CD는 이 새로운 `Application`들을 발견하고, 각각의 지시에 따라 해당 애플리케이션들의 소스코드 저장소를 찾아가 실제 워크로드를 배포하기 시작한다.

이제부터 플랫폼에 새로운 애플리케이션을 추가하고 싶다면, 우리는 더 이상 클러스터에 직접 접근할 필요가 없다. 그저 `my-platform-repo`의 `apps` 디렉토리에 새로운 `new-app.yaml` 파일을 추가하는 Pull Request를 생성하기만 하면 된다. PR이 병합되는 순간, `root-app`이 이 변화를 감지하여 `new-app`이라는 `Application`을 클러스터에 생성하고, 연쇄적으로 실제 애플리케이션이 배포되는 모든 과정이 자동으로 일어난다.

App of Apps 패턴은 복잡성을 계층적으로 관리하고, 플랫폼 전체의 상태를 단 하나의 Git 저장소에서 선언적으로 제어할 수 있게 해주는 GitOps의 정수다.

이제 우리는 명령형 관리의 혼돈에서 벗어나, 예측 가능하고, 감사 가능하며, 완전히 자동화된 방식으로 플랫폼을 운영할 수 있는 견고한 기반을 마련했다. 하지만 우리의 애플리케이션들은 아직 고립된 섬과 같다. 다음 장에서는 이 섬들을 서로 연결하고, 외부 세계와 소통하게 만드는 복잡하고도 매혹적인 쿠버네티스 네트워킹의 세계로 깊이 잠수해 볼 것이다.