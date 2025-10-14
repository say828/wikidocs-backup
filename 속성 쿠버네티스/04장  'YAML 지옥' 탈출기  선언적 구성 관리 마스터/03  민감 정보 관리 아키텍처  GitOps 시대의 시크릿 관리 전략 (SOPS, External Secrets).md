## 03\. 민감 정보 관리 아키텍처: GitOps 시대의 시크릿 관리 전략 (SOPS, External Secrets)

우리는 이제 커스터마이즈와 헬름이라는 강력한 무기를 손에 넣고 YAML 지옥을 정복할 준비를 마쳤다. 모든 구성은 Git 저장소라는 단일 진실 공급원(Single Source of Truth)에서 선언적으로 관리되며, Argo CD와 같은 GitOps 도구는 이 선언을 클러스터의 실제 상태와 끊임없이 동기화한다. 이 얼마나 아름다운 세상인가. 하지만 이 완벽해 보이는 그림에는 치명적인 구멍이 하나 있다. 바로 \*\*시크릿(Secret)\*\*이다.

데이터베이스 암호, 외부 서비스 API 키, TLS 인증서와 같은 민감 정보를 어떻게 할 것인가? 만약 당신이 이 값들을 평문(plain text) 그대로 YAML 파일에 담아 Git 저장소에 커밋하는 실수를 저지른다면, 당신은 플랫폼의 모든 현관문을 활짝 열어놓고 "가져갈 수 있는 만큼 가져가시오"라고 외치는 것과 같다. Git 저장소는 한번 커밋된 내용을 영원히 기억하며, 접근 권한이 있는 모든 개발자와 CI/CD 시스템에 그 비밀을 고스란히 노출시킨다.

쿠버네티스가 제공하는 기본 `Secret` 오브젝트는 그저 Base64로 인코딩된 문자열일 뿐, 암호화와는 거리가 멀다. `echo "supersecret" | base64`가 안전하다고 믿는 사람은 아무도 없을 것이다. 그렇다면 우리는 이 선언적 GitOps 패러다임의 우아함을 포기하고, 시크릿만은 수동으로 `kubectl create secret` 명령을 통해 관리해야 하는 것일까? 그것은 GitOps의 핵심 원칙을 무너뜨리는 패배적인 타협이다.

다행히도, 우리에게는 두 가지 현명한 대안이 있다. 이 두 가지 전략은 GitOps의 철학을 유지하면서도 민감 정보를 안전하게 다루기 위한, 오늘날 가장 널리 채택되는 실전 아키텍처다.

### 전략 1: 암호화된 시크릿을 Git에 저장하기 (Sealed Secrets, SOPS)

첫 번째 전략은 "그래도 모든 것은 Git 안에 있어야 한다"는 순수주의적 접근이다. 시크릿을 평문이 아닌, **강력한 암호화 알고리즘으로 잠근 상태**로 Git 저장소에 저장하는 방식이다. 이 암호화된 파일 자체는 더 이상 민감 정보가 아니므로 Git에 커밋해도 안전하다. 그리고 클러스터 내부에는 이 파일을 해독할 수 있는 열쇠를 가진 유일한 존재, 즉 특별한 컨트롤러를 배치한다.

이 방식을 구현하는 대표적인 도구가 바로 **모질라의 SOPS(Secrets OPerationS)** 다. SOPS는 YAML, JSON, ENV 파일 등의 값(value) 부분만을 AWS KMS, Google Cloud KMS, PGP 등과 같은 검증된 암호화 서비스를 이용해 암호화하는 간단하고도 강력한 CLI 도구다.

작동 방식은 다음과 같다.

1.  **암호화:** 플랫폼 엔지니어는 자신의 로컬 환경에서 KMS 키에 접근할 수 있는 권한을 이용해 `db-secrets.yaml` 파일의 `password` 필드를 암호화한다.

    ```bash
    # 암호화 전 (db-secrets.dec.yaml)
    apiVersion: v1
    kind: Secret
    metadata:
      name: db-credentials
    stringData:
      password: "supersecret-password"

    # SOPS로 암호화 실행
    sops --encrypt --kms <KMS_KEY_ARN> db-secrets.dec.yaml > db-secrets.enc.yaml 
    ```

    암호화된 `db-secrets.enc.yaml` 파일은 이제 안전하게 Git에 커밋할 수 있다.

2.  **복호화:** 클러스터에는 Argo CD와 같은 GitOps 컨트롤러가 이 암호화된 파일을 가져온다. 이때, Argo CD는 SOPS와 통합되도록 설정되어 있으며, 클러스터의 서비스 어카운트(Service Account)는 해당 KMS 키에 접근할 수 있는 IAM 권한을 부여받았다.

3.  **동기화:** Argo CD는 Git에서 가져온 파일을 클러스터에 적용하기 직전에, SOPS를 통해 실시간으로 복호화한다. 최종적으로 클러스터의 etcd에 저장되는 `Secret` 오브젝트는 평문 상태가 되지만, 이 모든 과정은 파이프라인 내부에서 안전하게 처리되며 Git 저장소에는 암호화된 버전만이 남게 된다.

[그림: SOPS를 활용한 GitOps 시크릿 관리 흐름도]

이 방식은 모든 구성을 Git에서 관리한다는 GitOps의 원칙을 가장 순수하게 지킬 수 있다는 장점이 있다.

### 전략 2: 외부 시크릿 저장소 참조하기 (External Secrets Operator)

두 번째 전략은 "시크릿은 애초에 Git에 저장하는 것이 아니다"라는 현실적인 접근이다. 대부분의 성숙한 조직은 이미 AWS Secrets Manager, HashiCorp Vault, Google Secret Manager와 같은 전문적인 중앙 시크릿 관리 시스템을 운영하고 있다. 이 전략은 Git 저장소에는 실제 비밀 값을 저장하는 대신, "내가 필요한 비밀은 Vault의 저 경로에 저장되어 있으니, 가져다줘"라는 \*\*참조 정보(reference)\*\*만을 기록하는 방식이다.

이 마법을 가능하게 하는 것이 바로 **External Secrets Operator(ESO)** 와 같은 컨트롤러다.

작동 방식은 다음과 같다.

1.  **참조 선언:** 우리는 `Secret`을 직접 만드는 대신, `ExternalSecret`이라는 새로운 종류의 커스텀 리소스(CRD)를 Git에 커밋한다. 이 리소스에는 실제 비밀 값이 아닌, 외부 저장소의 경로 정보만이 담겨있다.
    ```yaml
    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      name: db-credentials-es
    spec:
      secretStoreRef: # 어떤 외부 저장소를 사용할지 지정
        name: vault-backend
        kind: ClusterSecretStore
      target: #最终的に作成されるSecretオブジェクトの名前
        name: db-credentials # 최종적으로 생성될 Secret 오브젝트의 이름
      data:
      - secretKey: password # 생성될 Secret의 key
        remoteRef: # Vault에서의 경로
          key: secret/data/myapp/db
          property: password
    ```
2.  **동기화 루프:** 클러스터에 설치된 External Secrets Operator는 이 `ExternalSecret` 오브젝트의 생성을 감지한다.
3.  **가져오기 및 생성:** ESO는 `ClusterSecretStore`에 정의된 인증 정보를 이용해 외부 시크릿 저장소(예: Vault)에 직접 접근하여 해당 경로의 비밀 값을 안전하게 가져온다. 그리고 그 값을 담아, 우리가 최종적으로 원하는 형태의 **네이티브 쿠버네티스 `Secret` 오브젝트(`db-credentials`)를 클러스터 내부에 직접 생성**해준다.

[그림: External Secrets Operator가 외부 저장소와 통신하여 Secret을 생성하는 아키텍처]

이 방식의 가장 큰 장점은 시크릿의 생명주기를 애플리케이션의 배포와 완전히 분리하고, 조직의 중앙 보안 정책을 그대로 따를 수 있다는 점이다. 개발자는 더 이상 실제 비밀 값에 접근할 필요가 없으며, 보안팀은 중앙 저장소에서 모든 시크릿의 접근, 교체(rotation)를 통제할 수 있다.

어떤 전략을 선택할 것인가는 당신의 조직의 보안 정책과 기존 인프라에 달려있다. 중요한 것은, 어떤 방식을 선택하든 더 이상 평문 시크릿을 Git에 커밋하는 원죄를 저지르지 않아도 된다는 사실이다.

이로써 우리는 'YAML 지옥'이라 불리는 구성 관리의 혼돈을 정복하고, 심지어 가장 민감한 정보까지도 선언적으로 안전하게 다룰 수 있는 완전한 GitOps 워크플로우를 완성했다. 이제 우리의 플랫폼은 여러 환경에 걸쳐 일관되고, 재현 가능하며, 안전하다.

하지만 정적인 배포만으로는 충분하지 않다. 우리는 어떻게 하면 새로운 버전의 애플리케이션을 사용자 모르게, 단 한 순간의 중단도 없이 안전하게 릴리즈할 수 있을까? 다음 장에서는 롤링 업데이트의 한계를 넘어, 블루/그린, 카나리와 같은 고급 배포 전략의 세계로 떠나볼 것이다.