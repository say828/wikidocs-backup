## 02\. Helm: 쿠버네티스 애플리케이션 패키징과 재사용의 표준

커스터마이즈가 YAML의 차이를 관리하는 정교한 외과 의사의 메스라면, \*\*헬름(Helm)\*\*은 복잡한 애플리케이션 전체를 하나의 완전한 제품으로 조립하고 배포하는 거대한 조선소의 설계도와 같다. 헬름은 스스로를 "쿠버네티스를 위한 패키지 매니저"라고 칭한다. `apt`가 우분투의 패키지를, `npm`이 Node.js의 패키지를 관리하듯, 헬름은 쿠버네티스 애플리케이션의 복잡성을 관리하는 것을 목표로 한다.

헬름은 커스터마이즈와는 정반대의 철학에서 출발한다. 바로 \*\*템플릿(Templating)\*\*의 힘을 적극적으로 활용하는 것이다. 헬름의 세계에서, 당신의 `deployment.yaml` 파일은 더 이상 순수한 YAML이 아니다. 그것은 Go 템플릿 언어로 작성된, 동적으로 YAML을 생성하기 위한 하나의 \*\*'틀'\*\*이다.

### 헬름의 세 가지 핵심 개념

헬름의 세계를 이해하기 위해서는 세 가지 핵심 개념을 먼저 알아야 한다.

1.  **차트 (Chart):** 헬름의 패키지 형식이다. 애플리케이션을 구동하는 데 필요한 모든 쿠버네티스 리소스 템플릿, 기본 설정값, 그리고 의존성(예: PostgreSQL 데이터베이스 차트)에 대한 정보가 하나의 디렉토리 구조 안에 체계적으로 담겨 있다. 이것이 바로 재사용 가능한 애플리케이션의 '설계도 묶음'이다.
2.  **릴리즈 (Release):** 차트를 특정 설정값(Values)을 이용해 쿠버네티스 클러스터에 설치한 **실행 인스턴스**다. 예를 들어, 당신은 동일한 `wordpress` 차트를 이용해 '마케팅팀 블로그' 릴리즈와 '개발팀 블로그' 릴리즈를 동일한 클러스터에 독립적으로 설치하고 관리할 수 있다.
3.  **Values:** 차트의 템플릿에 주입될 설정값들이다. 차트 제작자는 `values.yaml` 파일에 기본값들을 정의해두고, 사용자는 차트를 설치하거나 업그레이드할 때 이 값들을 자신만의 값으로 **덮어쓸(override)** 수 있다. 이것이 헬름의 유연성의 핵심이다.

### 템플릿의 힘과 위험성

헬름의 템플릿이 어떻게 동작하는지 간단한 `deployment.yaml` 예시를 통해 살펴보자.

**`my-chart/templates/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }} # <-- values.yaml에서 값을 가져온다
  template:
    spec:
      containers:
      - name: my-app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.internalPort }}
        
        {{- if .Values.livenessProbe.enabled }} # <-- 조건문 로직
        livenessProbe:
          httpGet:
            path: /healthz
            port: {{ .Values.service.internalPort }}
        {{- end }}
```

**`my-chart/values.yaml` (기본값)**

```yaml
replicaCount: 1
image:
  repository: my-app
  tag: "1.0.0"
service:
  internalPort: 8080
livenessProbe:
  enabled: true
```

사용자는 이 차트를 설치할 때, `helm install my-release ./my-chart --set replicaCount=3 --set image.tag=1.5.0` 와 같이 명령줄에서 값을 직접 변경할 수 있다. 헬름은 이 값들을 템플릿과 조합하여 최종적으로 유효한 YAML을 생성한 뒤 클러스터로 보낸다.

이 템플릿 방식은 커스터마이즈가 할 수 없는 강력한 기능을 제공한다.

  * **프로그래밍 로직:** `if/else` 블록을 사용해 특정 `value`가 `true`일 때만 특정 리소스(예: ServiceMonitor)나 YAML 블록(예: livenessProbe)을 생성할 수 있다.
  * **반복문과 함수:** `range`를 사용해 값의 목록을 순회하며 여러 개의 컨테이너나 ConfigMap 데이터를 동적으로 생성할 수 있으며, 내장 함수와 헬퍼(helper) 템플릿을 통해 복잡한 문자열 처리나 로직을 재사용할 수 있다.
  * **의존성 관리:** 당신의 애플리케이션 차트가 `postgresql` 차트에 의존한다고 `Chart.yaml`에 명시하면, `helm install` 시 헬름이 알아서 PostgreSQL까지 함께 설치하고 관리해준다.

하지만 이 강력함에는 대가가 따른다. 템플릿이 복잡해질수록, 최종적으로 어떤 YAML이 생성될지 예측하고 디버깅하는 것이 매우 어려워진다. YAML의 들여쓰기와 템플릿 구문이 뒤섞여 가독성이 떨어지는 것은 물론, 잘못된 템플릿 로직은 유효하지 않은 YAML을 생성하여 배포를 실패로 이끈다.

### Kustomize vs. Helm: 언제 무엇을 쓸 것인가?

그렇다면 우리는 언제 커스터마이즈를, 언제 헬름을 선택해야 하는가?

[도표: Kustomize와 Helm의 핵심 철학 및 사용 사례 비교]
| 구분 | Kustomize | Helm |
| :--- | :--- | :--- |
| **핵심 철학** | **오버레이 (Overlaying)** | **템플릿 (Templating)** |
| **강점** | 단순함, 명확성, YAML 네이티브 | 패키징, 재사용성, 동적 구성, 의존성 관리 |
| **관리 대상** | \*\*구성(Configuration)\*\*의 차이 | **애플리케이션(Application)** 전체 패키지 |
| **주요 사용 사례** | • 내부 애플리케이션의 환경별 설정 관리<br>• GitOps 파이프라인에서의 최종 변형 | • 복잡한 오픈소스 앱(Prometheus, Argo CD) 설치<br>• 여러 팀/고객에게 앱을 패키지로 배포 |

**결론적으로, 둘은 경쟁자가 아니라 상호 보완적인 도구다.**

  * 당신이 직접 개발한 **내부 애플리케이션**의 개발/스테이징/운영 환경별 구성을 관리하는 것이 주된 목적이라면, **커스터마이즈**의 단순함과 명확성이 빛을 발한다.
  * Prometheus나 Argo CD와 같은 **외부의 복잡한 애플리케이션**을 클러스터에 설치하거나, 당신의 애플리케이션을 다른 팀이 쉽게 가져다 쓸 수 있도록 **패키징**하여 배포해야 한다면, **헬름**이 정답이다.

심지어 많은 고급 사용자는 이 둘을 함께 사용한다. 헬름을 이용해 외부 차트의 기본 YAML을 생성(render)한 뒤, 그 결과물 위에 커스터마이즈를 적용하여 우리 조직만의 미세한 설정을 덧씌우는 방식은 두 도구의 장점만을 취하는 현명한 전략이다.

이제 우리는 선언적으로 YAML의 복잡성을 관리하는 강력한 도구들을 손에 넣었다. 하지만 아직 해결되지 않은 가장 민감한 문제가 남아있다. 데이터베이스 암호, API 키와 같은 비밀 정보들을 어떻게 이 선언적인 GitOps 워크플로우 안에서 안전하게 관리할 수 있을까? 다음 절에서는 이 위험한 여정을 위한 아키텍처를 탐구해본다.