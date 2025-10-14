## KNative Serving: 트래픽 기반 Scale-to-Zero 및 리비전 관리

KNative 아키텍처의 첫 번째 기둥인 **KNative Serving**의 실제 동작을 살펴보겠습니다. Serving은 어떻게 우리의 컨테이너를 **트래픽 기반으로 0에서 N까지 자동으로 스케일링**하고, 어떻게 **안전한 버전 관리**를 가능하게 할까요?

이를 위해, "Hello, `<이름>`"을 반환하는 간단한 `greeter-service`를 KNative `Service`로 배포해 보겠습니다.

-----

### 1\. KNative `Service` (ksvc) 리소스 작성

개발자가 KNative와 상호작용하는 가장 기본적인 방법은 `Service` 리소스를 YAML 파일로 작성하는 것입니다. 이 파일 하나가 `Deployment`, `Service`, `HPA`, `Istio VirtualService` 등 수많은 쿠버네티스 오브젝트를 대체합니다.

```yaml
# greeter-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service # 1. 쿠버네티스 기본 Service가 아닌, KNative Service (ksvc)
metadata:
  name: greeter-service
spec:
  template: # 2. 이 서비스를 통해 배포될 Pod의 명세 (Deployment의 template과 유사)
    spec:
      containers:
        - image: ecommerce/greeter-service:v1 # 3. 실행할 컨테이너 이미지
          ports:
            - containerPort: 8080 # 컨테이너가 리스닝하는 포트
          env: # 환경 변수 설정
            - name: GREETING_PREFIX
              value: "Hello, "
```

이 YAML 파일은 "KNative야, `ecommerce/greeter-service:v1` 이미지를 실행하고, 외부 요청을 받을 수 있도록 준비해 줘"라는 매우 단순하고 선언적인 요청입니다.

-----

### 2\. 배포 및 Scale-to-Zero 확인

```bash
# KNative Service를 클러스터에 배포
kubectl apply -f greeter-service.yaml

# 배포된 ksvc 상태 확인
kubectl get ksvc

# NAME              URL                                           LATESTCREATED         LATESTREADY           READY   REASON
# greeter-service   http://greeter-service.default.example.com    greeter-service-00001   greeter-service-00001   True
```

배포가 완료되면 KNative는 서비스에 접근할 수 있는 고유한 `URL`을 자동으로 생성합니다. `kubectl get pods`를 실행하면, 요청을 처리하기 위한 Pod가 하나 생성된 것을 볼 수 있습니다.

#### Scale-to-Zero의 마법

이제 이 서비스에 아무런 요청도 보내지 않고 약 60초(기본값)를 기다려 봅시다. 그리고 다시 Pod 목록을 확인합니다.

```bash
# 60초 후...
kubectl get pods

# No resources found in default namespace.
```

**Pod가 사라졌습니다\!** 이것이 바로 **Scale-to-Zero**입니다. KNative의 오토스케일러(KPA, Knative Pod Autoscaler)는 트래픽이 없음을 감지하고, 리소스를 절약하기 위해 Pod의 개수를 0으로 자동으로 줄였습니다.

#### Scale-from-Zero (Cold Start)

이제 `curl`을 사용하여 서비스 URL로 첫 번째 요청을 보내 봅시다.

```bash
curl http://greeter-service.default.example.com?name=KNative
# (응답이 오기까지 몇 초 정도의 지연이 발생)
# --> Hello, KNative
```

첫 번째 요청은 Pod가 새로 생성되고 애플리케이션이 시작되어야 하므로 약간의 **콜드 스타트 지연**이 발생합니다. 하지만 `kubectl get pods`를 다시 실행해 보면, 요청을 처리하기 위해 새로운 Pod가 다시 생성된 것을 확인할 수 있습니다. 이후의 요청들은 이 '웜업'된 Pod로 전달되므로 매우 빠르게 응답합니다.

-----

### 3\. 리비전(Revision) 관리와 트래픽 분배

이제 `greeter-service`의 인사말을 바꾸고 `v2` 이미지를 빌드했다고 가정해 봅시다. `greeter-service.yaml` 파일의 이미지 태그만 `v2`로 변경하고 다시 적용합니다.

```yaml
# greeter-service.yaml
# ...
spec:
  template:
    spec:
      containers:
        - image: ecommerce/greeter-service:v2 # v1 -> v2로 변경
# ...
```

```bash
kubectl apply -f greeter-service.yaml
```

이 순간 KNative Serving은 다음과 같이 동작합니다.

1.  `spec.template`의 변경을 감지하고, 이 새로운 설정을 위한 \*\*불변(Immutable) 리비전 `greeter-service-00002`\*\*를 생성합니다. (기존 `00001` 리비전은 그대로 유지됩니다.)
2.  기본적으로, **모든(100%) 트래픽을 새로운 `00002` 리비전으로** 자동 전환합니다.

만약 `v2` 버전에 대한 카나리 배포를 하고 싶다면, `spec.traffic` 블록을 사용하여 트래픽을 정교하게 제어할 수 있습니다.

```yaml
# greeter-service-canary.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: greeter-service
spec:
  # ... (template은 v2 이미지로 설정) ...
  
  # 트래픽 분배 규칙을 명시적으로 정의
  traffic:
    - tag: new # 'new'라는 이름의 트래픽 타겟
      revisionName: greeter-service-00002 # v2 리비전으로
      percent: 10 # 10%의 트래픽을 보냄
    - tag: old
      revisionName: greeter-service-00001 # v1 리비전으로
      percent: 90 # 90%의 트래픽을 보냄
```

이 YAML을 적용하면, KNative는 내부적으로 Istio `VirtualService`를 제어하여, `greeter-service`로 오는 트래픽의 10%를 `v2` 리비전으로, 90%를 `v1` 리비전으로 분배합니다.

**결론적으로,** KNative Serving은 `Deployment`, `HPA`, `Service`, `VirtualService` 등 복잡한 쿠버네티스 리소스들을 **`ksvc`라는 단일 리소스로 추상화**하여, 개발자에게 매우 단순화된 서버리스 경험을 제공합니다. 이를 통해 우리는 자동화된 Scale-to-Zero와 안전한 리비전 기반의 트래픽 관리를 손쉽게 구현할 수 있습니다.