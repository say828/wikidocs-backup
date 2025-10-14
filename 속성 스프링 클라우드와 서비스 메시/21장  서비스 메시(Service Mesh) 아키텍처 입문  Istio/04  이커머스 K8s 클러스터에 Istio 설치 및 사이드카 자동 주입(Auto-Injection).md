## 이커머스 K8s 클러스터에 Istio 설치 및 사이드카 자동 주입(Auto-Injection)

이제 이론을 넘어, 우리 이커머스 애플리케이션이 실행 중인 쿠버네티스 클러스터에 **Istio를 실제로 설치**하고, 우리의 마이크로서비스들을 서비스 메시에 '등록'하는 과정을 진행해 보겠습니다.

-----

### 1\. Istio 설치 도구: `istioctl`

Istio를 설치하는 방법에는 여러 가지가 있지만(Helm 차트 등), 공식적으로 권장되는 가장 간단하고 강력한 방법은 Istio 전용 CLI 도구인 \*\*`istioctl`\*\*를 사용하는 것입니다. `istioctl`은 설치뿐만 아니라, 설치 프로파일 관리, 업그레이드, 디버깅 등 Istio 클러스터를 관리하는 데 필요한 모든 기능을 제공합니다.

#### 1-1. `istioctl` 다운로드 및 설치

먼저, 로컬 머신에 `istioctl` CLI를 설치합니다.

```bash
# istioctl 다운로드 및 압축 해제 (최신 버전 기준)
curl -L https://istio.io/downloadIstio | sh -

# istioctl 바이너리를 PATH에 추가
cd istio-*/bin
export PATH=$PWD:$PATH
```

#### 1-2. 클러스터에 Istio 컨트롤 플레인 설치

`istioctl install` 명령어는 **프로파일(Profile)** 기반으로 Istio를 설치합니다. 프로파일은 특정 목적에 맞게 미리 정의된 설정들의 묶음입니다. (예: `default`, `demo`, `minimal`, `empty`)

우리는 학습과 테스트에 유용한 모든 컴포넌트가 포함된 **`demo` 프로파일**을 사용하여 설치하겠습니다.

```bash
# 'demo' 프로파일로 Istio 설치를 진행
istioctl install --set profile=demo -y
```

이 명령어 하나로, `istioctl`은 우리 클러스터에 `istio-system`이라는 네임스페이스를 생성하고, 그 안에 `istiod` 컨트롤 플레인 Deployment와 관련 Service, Webhook 설정 등을 모두 자동으로 설치합니다.

#### 1-3. 설치 확인

설치가 완료되면, `istio-system` 네임스페이스에서 `istiod` Pod가 정상적으로 실행 중인지 확인합니다.

```bash
kubectl get pods -n istio-system

# NAME                            READY   STATUS    RESTARTS   AGE
# istiod-5f5d6f6f88-abcde         1/1     Running   0          2m
```

-----

### 2\. 사이드카 자동 주입 (Automatic Sidecar Injection)

이제 Istio 컨트롤 플레인은 준비되었습니다. 어떻게 우리의 `member-service`나 `order-service` Pod에 Envoy 사이드카 프록시를 '주입'할 수 있을까요? `Deployment` YAML 파일을 하나하나 수정해야 할까요?

Istio는 이 과정을 **자동으로** 처리하는 매우 우아한 방법을 제공합니다. 바로 쿠버네티스의 **Mutating Admission Webhook** 기능을 이용한 **자동 사이드카 주입**입니다.

**동작 방식:**

1.  우리는 사이드카를 주입하고 싶은 **네임스페이스(Namespace)**(예: `default` 네임스페이스)에 "Istio야, 이 공간을 지켜봐 줘"라는 의미의 \*\*레이블(Label)\*\*을 붙입니다.
2.  이제부터 해당 네임스페이스에 새로운 Pod가 생성될 때마다, 쿠버네티스 API 서버는 Pod의 명세(YAML)를 `istiod`의 Webhook으로 보냅니다.
3.  `istiod`는 이 원본 Pod 명세를 받아, **Envoy 사이드카 컨테이너와 트래픽 리다이렉션을 위한 `initContainer` 설정을 명세에 추가**하여 '수정된(Mutated)' Pod 명세를 다시 API 서버로 돌려줍니다.
4.  API 서버는 최종적으로 이 '수정된' 명세에 따라 Pod를 생성합니다.

이 모든 과정은 개발자에게 완벽하게 투명합니다. 개발자는 자신의 `Deployment` YAML에 오직 애플리케이션 컨테이너만 신경 쓰면 되고, 플랫폼(Istio)이 알아서 필요한 인프라(사이드카)를 주입해 줍니다.

-----

### 자동 주입 활성화 및 확인

#### 2-1. 네임스페이스에 레이블 추가

`default` 네임스페이스에 자동 주입을 활성화하는 레이블을 추가합니다.

```bash
kubectl label namespace default istio-injection=enabled
```

#### 2-2. 애플리케이션 재배포

이 설정은 **새로 생성되는 Pod**에만 적용됩니다. 따라서, 이미 실행 중인 `member-service`에 사이드카를 주입하려면, `rollout restart` 명령어로 Pod를 새로 생성해야 합니다.

```bash
# member-service Deployment의 모든 Pod를 점진적으로 재시작
kubectl rollout restart deployment member-service
```

#### 2-3. 주입 확인

잠시 후, 새로 생성된 `member-service` Pod의 상태를 확인합니다.

```bash
kubectl get pods

# NAME                              READY   STATUS    RESTARTS   AGE
# member-service-6d7b8c8f9b-fghij   2/2     Running   0          30s
# member-service-6d7b8c8f9b-klmno   2/2     Running   0          35s
```

`READY` 컬럼이 `1/1`이 아닌 \*\*`2/2`\*\*로 표시되는 것을 볼 수 있습니다. 이는 하나의 Pod 안에 **애플리케이션 컨테이너**와 **istio-proxy(Envoy) 컨테이너** 두 개가 모두 성공적으로 실행 중임을 의미합니다.

이것으로 21장을 마칩니다. 우리는 성공적으로 Istio 컨트롤 플레인을 설치하고, 우리의 마이크로서비스들을 서비스 메시의 데이터 플레인에 편입시켰습니다. 이제 우리의 애플리케이션은 Istio가 제공하는 강력한 트래픽 제어, 안정성, 보안 기능을 적용받을 준비가 되었습니다. 다음 22장에서는 이 기능들을 실제로 활용해 보겠습니다.