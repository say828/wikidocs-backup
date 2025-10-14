## 01\. Kustomize: YAML을 프로그래밍 방식으로 다루는 기술

ConfigMap이 해결해주지 못한 문제, 즉 YAML 구조 자체의 '차이'를 관리하는 것에 대한 쿠버네티스다운 첫 번째 대답이 바로 \*\*커스터마이즈(Kustomize)\*\*다. 커스터마이즈는 쿠버네티스 커뮤니티에서 태어나 `kubectl`에 기본적으로 통합된, 말 그대로 'YAML 네이티브' 구성 관리 도구다.

커스터마이즈의 철학은 놀랍도록 단순하면서도 강력하다. **"템플릿을 사용하지 말라(template-free)."**

다른 많은 도구들이 YAML 파일에 `{{ .Values.replicaCount }}`와 같은 변수를 심어놓고 나중에 값을 채워 넣는 **템플릿(Templating)** 방식을 사용하는 것과 달리, 커스터마이즈는 원본 YAML 파일을 **절대로 건드리지 않는다.** 대신, 잘 만들어진 공통의 **베이스(base)** YAML 위에, 각 환경의 고유한 변경 사항만을 \*\*패치(patch)\*\*처럼 덧씌우는 **오버레이(overlay)** 방식을 사용한다.

템플릿 엔진이 원본 설계도에 구멍을 뚫어놓고 나중에 채워 넣는 방식이라면, 커스터마이즈는 잘 만들어진 원본 설계도 위에 투명한 오버레이 필름을 덧대어 변경 사항만 그려 넣는 방식과 같다. 원본은 언제나 깨끗하게 유지되며, 우리는 오직 '차이'에만 집중할 수 있다.

이것이 어떻게 동작하는지 구체적인 디렉토리 구조를 통해 살펴보자.

```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   └── replicas-patch.yaml
    └── production/
        ├── kustomization.yaml
        └── resources-patch.yaml
```

**1. `base` 디렉토리: 공통의 기반**

이곳에는 모든 환경에서 공통적으로 사용될 표준 YAML 파일들이 위치한다. 여기에는 환경별 변수가 전혀 없다. 가장 순수하고 기본적인 애플리케이션의 정의다.

**`base/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1 # 기본값은 1개
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:1.0.0 # 기본 이미지 태그
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
```

그리고 이 디렉토리의 리소스들을 알려주는 `kustomization.yaml` 파일이 있다.

**`base/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

**2. `overlays` 디렉토리: 환경별 차이점**

이제 마법이 시작되는 곳이다. 프로덕션 환경(`overlays/production`)을 위한 구성을 만들어보자. 우리는 `replicas`를 5개로 늘리고, 자원 요청량을 더 높게 설정하고 싶다. 이를 위해 우리는 거대한 `deployment.yaml`을 복사하는 대신, **변경할 부분만을 담은 작은 패치 파일**을 만든다.

**`overlays/production/resources-patch.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app # <-- 어떤 리소스를 수정할지 알려주는 키
spec:
  replicas: 5 # <-- 변경할 값 1
  template:
    spec:
      containers:
      - name: my-app
        resources: # <-- 변경할 값 2
          requests:
            cpu: "1"
            memory: "2Gi"
```

그리고 프로덕션 오버레이의 `kustomization.yaml` 파일에서 이 모든 것을 조립하도록 지시한다.

**`overlays/production/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases: # 어느 base를 사용할지 지정
- ../../base

patchesStrategicMerge: # 어떤 패치를 덧씌울지 지정
- resources-patch.yaml

images: # 이미지 태그를 변경하는 더 쉬운 방법
- name: my-app
  newTag: "1.5.0-prod"
```

이제 모든 준비가 끝났다. 다음 명령어를 실행하면 어떤 일이 벌어지는지 확인해보자.

```bash
kubectl kustomize overlays/production
```

커스터마이즈는 `base` 디렉토리의 `deployment.yaml`과 `service.yaml`을 읽어온 뒤, 그 위에 `overlays/production`에 정의된 `replicas`, `resources`, `image tag` 변경 사항을 솜씨 좋게 덧씌워 최종적으로 완성된 프로덕션용 YAML을 화면에 출력해준다.

이제 우리는 더 이상 중복된 YAML과 씨름할 필요가 없다. 모든 공통 설정은 `base`에 있고, 각 환경의 고유성은 `overlays` 디렉토리 안에서 명확하게 관리된다. `kubectl apply -k overlays/production` 명령 하나로, 우리는 복잡한 템플릿 언어나 스크립트 없이도 완벽하게 환경별 구성을 클러스터에 적용할 수 있다.

커스터마이즈는 YAML의 구조를 이해하고, 그 차이점을 선언적으로 관리하는 가장 쿠버네티스다운 방식이다. 하지만 커스터마이즈가 만병통치약은 아니다. 만약 당신의 애플리케이션이 Redis나 PostgreSQL 같은 의존성을 가지고 있고, 이 모든 것을 하나의 완전한 '패키지'로 묶어 다른 팀이나 커뮤니티에 배포하고 싶다면 어떻게 해야 할까? 혹은, 특정 조건에 따라 아예 리소스 자체를 생성하거나 생성하지 않는 등의 좀 더 복잡한 로직이 필요하다면?

이때 우리는 쿠버네티스 생태계의 또 다른 거인, 패키지 매니저의 표준인 헬름(Helm)을 만나게 된다.