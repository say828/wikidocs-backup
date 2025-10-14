## 오토스케일링(HPA): 트래픽에 따른 Pod 자동 확장/축소

16장 03절까지 우리는 쿠버네티스를 통해 장애로부터 스스로를 복구하는 '자가 치유' 시스템을 만들었습니다. 하지만 '안정성'만으로는 부족합니다. 비즈니스는 '효율성' 또한 요구합니다.

우리는 `Deployment` YAML에 `replicas: 2`라고 복제본의 개수를 '고정'했습니다.

  * **문제점 1 (피크 타임):** 갑자기 마케팅 이벤트가 성공하여 트래픽이 10배로 폭증하면, 2개의 Pod는 과부하에 걸려 응답이 느려지고 결국 장애로 이어질 것입니다.
  * **문제점 2 (새벽 시간):** 트래픽이 거의 없는 새벽 3시에도, 2개의 Pod는 계속 실행되며 클라우드 비용을 낭비하고 있습니다.

운영자가 24시간 내내 트래픽을 지켜보며 수동으로 `kubectl scale deployment --replicas=10` 같은 명령어를 실행할 수는 없습니다. 이 문제를 해결하는 쿠버네티스의 자동화된 솔루션이 바로 \*\*HPA(Horizontal Pod Autoscaler)\*\*입니다.

-----

### HPA: 똑똑한 슈퍼마켓 매니저 🧑‍💼

HPA는 **관찰된 메트릭(주로 CPU나 메모리 사용량)을 기반으로, Deployment가 관리하는 Pod의 개수를 자동으로 늘리거나 줄이는** 쿠버네티스 컨트롤러입니다.

**슈퍼마켓 매니저 비유:**

  * **매니저:** HPA 컨트롤러
  * **계산대:** Pod
  * **고객 줄 길이:** CPU 사용량 (메트릭)

현명한 매니저는 계산대를 고정된 숫자로 운영하지 않습니다. 대신, \*\*고객 줄(CPU 사용량)\*\*을 계속 지켜봅니다.

  * 손님이 몰려 줄이 길어지면 (`평균 CPU 사용량 > 80%`), 휴게실에 있던 다른 계산원들을 호출하여 계산대를 추가로 엽니다 (`Pod 개수 증가`).
  * 손님이 없어 계산원들이 쉬고 있으면 (`평균 CPU 사용량 < 20%`), 일부 계산원들을 퇴근시켜 인건비를 절약합니다 (`Pod 개수 감소`).

-----

### HPA의 동작 원리

1.  **메트릭 수집:** HPA는 \*\*메트릭 서버(Metrics Server)\*\*라는 클러스터 애드온(Add-on)을 통해 각 Pod의 현재 CPU 및 메모리 사용량을 주기적으로 수집합니다.

2.  **상태 비교:** HPA 컨트롤러는 `Deployment`에 속한 모든 Pod들의 평균 CPU 사용량(**현재 값**)과, 우리가 HPA 설정에 정의한 **목표 값**을 비교합니다.

3.  **복제본 수 계산:** 다음 공식에 따라 '원하는 Pod 개수'를 계산합니다.
    `원하는 Pod 수 = ceil[ 현재 Pod 수 * ( 현재 CPU 사용량 / 목표 CPU 사용량 ) ]`

4.  **스케일링 실행:** HPA는 자신이 제어하는 `Deployment`의 `replicas` 필드 값을 방금 계산한 '원하는 Pod 수'로 업데이트합니다. 그러면 `Deployment`의 조정 루프(Reconciliation Loop)가 이 변경을 감지하고, Pod를 생성하거나 종료하여 원하는 상태를 맞춥니다.

-----

### HPA 적용하기

#### 1단계 (선행 작업): 메트릭 서버 설치

HPA가 동작하려면 클러스터에 메트릭 서버가 반드시 설치되어 있어야 합니다. (일반적으로 클라우드 제공업체의 K8s 서비스에는 기본 설치되어 있거나 쉽게 추가할 수 있습니다.)

```bash
# 메트릭 서버를 설치하는 일반적인 명령어
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### 2단계 (필수): Deployment에 Resource 요청 설정

HPA가 'CPU 사용률(%)'을 계산하려면, '100%'의 기준이 되는 \*\*'CPU 요청량(Request)'\*\*을 알아야 합니다. `Deployment` YAML에 각 컨테이너가 필요로 하는 최소한의 CPU 자원을 명시해야 합니다.

```yaml
# member-service-deployment.yaml
# ...
spec:
  template:
    spec:
      containers:
        - name: member-service-container
          image: ecommerce/member-service:0.0.1
          # --- HPA를 위한 리소스 요청량 설정 (필수!) ---
          resources:
            requests:
              cpu: "250m" # CPU 0.25 코어를 요청
              # memory: "512Mi" # 메모리 기반 스케일링도 가능
```

  * `250m`은 '250 millicores'를 의미하며, 이는 CPU 코어 1개의 1/4에 해당합니다.

#### 3단계: HPA 오브젝트 생성

`member-service`를 위한 HPA 규칙을 `YAML` 파일로 정의합니다.

```yaml
# member-service-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: member-service-hpa
spec:
  # 1. 스케일링할 대상(Deployment)을 지정
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: member-service
  
  # 2. Pod 개수의 최소/최대 범위 지정
  minReplicas: 2
  maxReplicas: 10
  
  # 3. 스케일링의 기준이 될 메트릭 정의
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          # 4. 모든 Pod의 평균 CPU 사용량이 80%를 초과하면 Pod를 늘림
          averageUtilization: 80
```

#### 적용 및 확인

```bash
# HPA 오브젝트를 클러스터에 적용
kubectl apply -f member-service-hpa.yaml

# HPA 상태 확인
kubectl get hpa
```

`kubectl get hpa` 명령어는 현재 CPU 사용량(TARGETS)과 복제본 수(REPLICAS)를 실시간으로 보여줍니다. 이제 부하 테스트 도구를 사용하여 `member-service`에 트래픽을 가하면, `REPLICAS` 수가 `minReplicas`인 2에서 `maxReplicas`인 10까지 자동으로 늘어났다가, 부하가 사라지면 다시 2로 줄어드는 것을 확인할 수 있습니다.

HPA는 진정한 클라우드 네이티브의 '탄력성(Elasticity)'을 우리 애플리케이션에 부여하는 핵심 기능입니다. 이를 통해 우리는 피크 타임의 성능과 평시의 비용 효율성을 모두 자동으로 확보할 수 있게 됩니다.