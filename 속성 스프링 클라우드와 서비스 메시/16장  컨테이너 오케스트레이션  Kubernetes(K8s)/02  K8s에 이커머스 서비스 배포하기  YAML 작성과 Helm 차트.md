## K8s에 이커머스 서비스 배포하기: YAML 작성과 Helm 차트

16장 00절에서 쿠버네티스의 핵심 오브젝트들을 배웠습니다. 이제 이 '설계도'들을 **YAML(YAML Ain't Markup Language)** 파일 형식으로 직접 작성하여, 15장에서 컨테이너화한 우리의 `member-service`를 실제 쿠버네티스 클러스터에 배포해 보겠습니다.

-----

### 1\. 순수한 YAML(Raw YAML)로 배포하기

가장 기본적인 방법은 각 오브젝트에 대한 YAML 파일을 하나씩 작성하여 `kubectl`이라는 쿠버네티스 CLI 도구로 클러스터에 적용하는 것입니다.

#### `Deployment` YAML 작성

`member-service` 컨테이너를 2개 실행하고, 장애 시 자동으로 복구하도록 `Deployment`를 정의합니다.

```yaml
# member-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: member-service
spec:
  replicas: 2 # 1. 이 Pod를 2개 복제하여 실행
  selector:
    matchLabels:
      app: member-service # 2. 이 Deployment가 관리할 Pod를 찾는 '꼬리표'
  template: # 3. 생성할 Pod의 설계도
    metadata:
      labels:
        app: member-service # 4. Pod에 'app: member-service' 꼬리표를 붙임
    spec:
      containers:
        - name: member-service-container
          # 5. 15장에서 빌드한 컨테이너 이미지
          image: ecommerce/member-service:0.0.1 
          ports:
            - containerPort: 8081 # 컨테이너가 노출하는 포트
```

#### `Service` YAML 작성

위 `Deployment`가 생성한 2개의 `member-service` Pod에 대한 안정적인 단일 진입점을 제공하기 위해 `Service`를 정의합니다.

```yaml
# member-service-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: member-service # 1. 이 서비스의 고유한 DNS 이름이 됨
spec:
  selector:
    app: member-service # 2. 'app: member-service' 꼬리표가 붙은 Pod들을 찾아서 묶음
  ports:
    - protocol: TCP
      port: 80 # 3. 클러스터 내부에서 이 Service가 노출할 포트
      targetPort: 8081 # 4. Service가 트래픽을 전달할 Pod 내부 컨테이너의 포트
  type: ClusterIP # 5. 클러스터 내부에서만 접근 가능한 타입 (기본값)
```

#### 클러스터에 적용 및 확인

작성한 YAML 파일들을 `kubectl apply` 명령어로 클러스터에 적용합니다.

```bash
# 1. Deployment 적용
kubectl apply -f member-service-deployment.yaml

# 2. Service 적용
kubectl apply -f member-service-service.yaml

# 3. 배포 상태 확인
kubectl get deployments # member-service가 Ready 상태인지 확인
kubectl get pods       # member-service Pod 2개가 Running 상태인지 확인
kubectl get services   # member-service가 ClusterIP를 할당받았는지 확인
```

-----

### 2\. 순수한 YAML의 문제점과 Helm의 등장

`member-service` 하나를 배포하는 데 벌써 2개의 YAML 파일이 필요했습니다. 우리 시스템은 10개 이상의 서비스와 DB, Kafka 등으로 구성될 것입니다. 수십 개의 YAML 파일을 수동으로 관리하는 것은 곧 재앙이 됩니다.

  * **엄청난 중복:** `member-service`의 Deployment YAML과 `order-service`의 Deployment YAML은 95% 동일하고, `image` 이름과 `port` 정도만 다를 것입니다.
  * **관리의 어려움:** '이커머스' 애플리케이션 전체를 하나의 단위로 설치, 업그레이드, 롤백, 삭제하는 것이 매우 어렵습니다.

이 문제를 해결하기 위해 등장한 것이 바로 **Helm: 쿠버네티스를 위한 패키지 관리자**입니다.

-----

### 3\. Helm 차트(Chart)로 애플리케이션 패키징하기

`Helm`은 리눅스의 `apt`나 `yum`, macOS의 `Homebrew`와 같습니다. 애플리케이션을 배포하는 데 필요한 모든 쿠버네티스 YAML 파일들과 설정을 \*\*'차트(Chart)'\*\*라는 하나의 패키지로 묶어 관리합니다.

  * **템플릿 (Templates):** 차트 안의 YAML 파일들은 **템플릿**으로 만들어져 있습니다. `replicas` 수, `image` 태그 등을 변수로 처리할 수 있습니다.
  * **값 (Values):** `values.yaml` 파일에 이 템플릿에 주입할 **변수 값**들을 정의합니다.
  * **릴리즈 (Release):** 하나의 차트를 특정 값(Values)으로 클러스터에 설치한 '실행 인스턴스'입니다.

#### Helm 템플릿 예시

`deployment.yaml`이 Helm 템플릿으로 어떻게 바뀌는지 살펴봅시다.

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  # 1. replicas 수를 values.yaml 파일에서 가져옴
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          # 2. 이미지 정보도 values.yaml 파일에서 조합
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

#### `values.yaml` 예시

이제 우리는 이 `values.yaml` 파일만 수정하면, YAML 템플릿을 건드리지 않고도 배포 설정을 쉽게 변경할 수 있습니다.

```yaml
# values.yaml
replicaCount: 2

image:
  repository: ecommerce/member-service
  tag: 0.0.1

service:
  port: 80
  targetPort: 8081
```

`order-service`를 배포하고 싶다면, `image.repository` 값만 `ecommerce/order-service`로 변경한 새로운 `values.yaml` 파일을 사용하면 됩니다.

#### Helm 명령어

이제 수십 개의 `kubectl apply` 명령어 대신, 단 하나의 `helm` 명령어로 전체 애플리케이션을 관리할 수 있습니다.

```bash
# 'my-ecommerce'라는 이름으로 'ecommerce-chart'를 설치
helm install my-ecommerce ./ecommerce-chart

# 이미지 태그를 0.0.2로 변경하여 업그레이드
helm upgrade my-ecommerce ./ecommerce-chart --set image.tag=0.0.2

# 전체 애플리케이션 삭제
helm uninstall my-ecommerce
```

**결론적으로,** 순수한 YAML은 쿠버네티스의 기본 동작을 이해하는 데 필수적이지만, 실제 프로덕션 환경에서는 **Helm**을 사용하여 애플리케이션을 '패키지'로 관리하는 것이 압도적으로 효율적이고 안전합니다. Helm은 복잡한 MSA 배포를 표준화하고 자동화하는 사실상의 업계 표준 도구입니다.