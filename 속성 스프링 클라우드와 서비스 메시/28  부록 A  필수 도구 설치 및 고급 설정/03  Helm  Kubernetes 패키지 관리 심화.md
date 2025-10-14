## Helm: Kubernetes 패키지 관리 심화

16장에서 우리는 `Helm`을 '쿠버네티스를 위한 패키지 관리자'라고 소개하며, 복잡한 YAML 파일들을 \*\*'차트(Chart)'\*\*라는 단위로 묶어 애플리케이션의 설치, 업그레이드, 삭제를 단순화하는 방법을 배웠습니다.

부록에서는 한 걸음 더 나아가, 최고의 실무자가 복잡한 프로덕션 환경을 관리하기 위해 반드시 알아야 할 Helm의 세 가지 고급 기능을 심층적으로 다룹니다.

-----

### 1\. 의존성 관리 (Subcharts)

우리 `order-service`는 애플리케이션 컨테이너만 배포한다고 해서 동작하지 않습니다. 반드시 **PostgreSQL 데이터베이스**와 **Redis 캐시**를 필요로 합니다. 그렇다면 `order-service`를 배포할 때마다, 운영자가 수동으로 PostgreSQL Helm 차트와 Redis Helm 차트를 별도로 설치해야 할까요?

Helm은 **차트 의존성(Dependencies)** 기능을 통해 이 문제를 우아하게 해결합니다. 즉, 우리 `order-service` 차트가 `postgresql`과 `redis` 차트에 '의존'한다고 명시할 수 있습니다.

`order-service`의 `Chart.yaml` 파일에 `dependencies` 블록을 추가합니다.

```yaml
# order-service/Chart.yaml
apiVersion: v2
name: order-service
description: A Helm chart for the e-commerce Order Service
version: 0.1.0

dependencies:
  # 1. PostgreSQL 차트에 대한 의존성 정의
  - name: postgresql
    # Bitnami 차트 저장소의 14.x.x 버전을 사용
    version: "14.x.x" 
    repository: "https://charts.bitnami.com/bitnami"
    # 2. (선택적) 이 하위 차트의 설정을 우리 values.yaml에서 덮어쓸 때 사용할 별칭(alias)
    alias: database
    # 3. 이 의존성이 비활성화될 조건을 명시 (예: 외부 DB를 사용할 경우)
    condition: database.enabled

  # 4. Redis 차트에 대한 의존성 정의
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

이제 차트 디렉토리에서 `helm dependency update` 명령어를 실행하면, Helm은 명시된 `postgresql`과 `redis` 차트를 `charts/` 하위 디렉토리로 다운로드합니다.

그 후 `helm install my-order ./order-service`를 실행하면, Helm은 **`order-service`뿐만 아니라, 의존성으로 포함된 PostgreSQL과 Redis까지 하나의 릴리즈로 묶어 원자적으로 설치**합니다. 이제 `order-service`는 자신만의 완전한 '패키지'가 되었습니다.

-----

### 2\. 템플릿 렌더링 및 디버깅 (`helm template`, `--dry-run`)

Helm 차트는 `values.yaml`과 템플릿 로직이 결합되어 최종 쿠버네티스 YAML을 생성합니다. 만약 `values.yaml`을 잘못 작성하여 문법 오류가 있는 YAML이 생성된다면, `helm install`은 실패할 것입니다.

배포를 실행하기 **전에**, Helm이 생성할 최종 YAML을 미리 확인하고 디버깅하는 것은 매우 중요합니다.

  * **`helm template`:** 클러스터에 접속하지 않고, 순수하게 로컬에서 최종 렌더링된 YAML을 표준 출력(stdout)으로 보여줍니다.

```bash
# my-order라는 릴리즈 이름으로 ./order-service 차트를 렌더링하여
# final-manifest.yaml 파일로 저장
helm template my-order ./order-service > final-manifest.yaml
```
   
  * **`helm install --dry-run`:** `template`과 유사하지만, 실제로 클러스터 API 서버와 통신하여 생성된 YAML이 유효한지 **서버 사이드 유효성 검사**까지 수행합니다. 실제 리소스를 생성하지만 않을 뿐, 설치 과정 전체를 시뮬레이션합니다.

```bash
# 실제 설치는 하지 않고, "만약 설치한다면" 어떤 결과가 나올지 시뮬레이션
helm install my-order ./order-service --dry-run
```

이 두 명령어는 CI/CD 파이프라인에서 배포 전 유효성을 검증하거나, 복잡한 템플릿 로직을 디버깅할 때 필수적입니다.

-----

### 3\. 배포 생명주기 관리 (Helm Hooks)

애플리케이션을 업그레이드할 때, 새로운 버전의 Pod를 띄우기 **전에** 반드시 **데이터베이스 스키마 마이그레이션** 작업이 먼저 실행되어야 하는 경우가 있습니다.

**Helm Hooks**는 `install`, `upgrade`, `delete`와 같은 릴리즈 생명주기의 특정 시점에 지정된 쿠버네티스 리소스(주로 `Job`)를 실행할 수 있게 해주는 강력한 메커셔니즘입니다.

`order-service`의 DB 마이그레이션을 위한 `Job` 템플릿에 훅(Hook) 어노테이션을 추가해 봅시다.

```yaml
# templates/db-migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migration
  annotations:
    # 1. 이 Job이 "helm hook"임을 명시
    "helm.sh/hook": pre-upgrade,pre-install
    # 2. 훅 실행 순서 (낮을수록 먼저 실행)
    "helm.sh/hook-weight": "-5"
    # 3. 훅 작업이 완료된 후, 해당 Job 리소스를 삭제하는 정책
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: db-migrator
          image: ecommerce/db-migrator:0.0.1 # Flyway/Liquibase가 포함된 마이그레이션 이미지
          # (DB 접속 정보 등은 Secret에서 가져옴)
      restartPolicy: Never
```

이제 운영자가 `helm upgrade my-order ./order-service`를 실행하면, Helm은 다음과 같이 동작합니다.

1.  가장 먼저 `pre-upgrade` 훅으로 지정된 `db-migration` Job을 실행합니다.
2.  이 Job이 성공적으로 완료될 때까지 **기다립니다.**
3.  Job이 성공하면, 그제서야 `order-service` Deployment의 롤링 업데이트 등 나머지 리소스들에 대한 업그레이드를 진행합니다.

Helm Hooks를 통해 우리는 복잡한 배포 절차를 안정적이고 자동화된 방식으로 Helm 차트 안에 모두 코드화(Codify)할 수 있습니다.

**결론적으로,** Helm은 단순한 템플릿 도구를 넘어, 의존성 관리, 디버깅, 생명주기 제어 등 애플리케이션 배포에 필요한 모든 것을 관리하는 성숙한 '패키지 관리' 생태계를 제공합니다.