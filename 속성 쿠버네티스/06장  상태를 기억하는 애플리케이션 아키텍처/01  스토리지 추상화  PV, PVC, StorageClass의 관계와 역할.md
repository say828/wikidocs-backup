## 01\. 스토리지 추상화: PV, PVC, StorageClass의 관계와 역할

우리는 스테이트풀셋이 각 파드에게 고유하고 영속적인 스토리지를 제공한다고 말했다. 하지만 잠깐, 여기서 한 걸음 물러서서 근본적인 질문을 던져보자. '스토리지'란 무엇인가? 그것은 AWS의 EBS 볼륨인가, 구글 클라우드의 Persistent Disk인가, 아니면 당신의 데이터센터에 있는 NFS 서버의 공유 디렉토리인가?

만약 당신의 파드 명세에 `awsElasticBlockStore: { volumeID: 'vol-012345' }` 와 같은 특정 인프라의 세부 정보를 직접 기입한다면, 당신은 쿠버네티스가 제공하는 가장 위대한 가치 중 하나인 \*\*'이식성(Portability)'\*\*을 스스로 내던지는 것이다. 그 애플리케이션은 더 이상 AWS 없이는 동작할 수 없는 반쪽짜리 클라우드 네이티브가 되고 만다. 애플리케이션은 자신이 필요로 하는 스토리지의 '속성'(예: "나는 10GB의 빠른 SSD가 필요해")에 대해서만 알아야 하며, 그 스토리지가 '어떻게' 구현되었는지에 대해서는 무지해야 한다.

이러한 **관심사의 분리**를 위해, 쿠버네티스는 스토리지에 대해 지극히 정교하고 아름다운 3단계 추상화 계층을 만들었다. 바로 **영속 볼륨 클레임(PersistentVolumeClaim, PVC)**, **영속 볼륨(PersistentVolume, PV)**, 그리고 \*\*스토리지클래스(StorageClass)\*\*다. 이 세 개념의 관계를 이해하는 것은 상태 저장 애플리케이션을 운영하는 모든 아키텍트의 기본 소양이다.

-----

### PVC와 PV: 사용자와 공급자의 분리

이 관계를 이해하는 가장 좋은 비유는 '식당의 주문'이다.

  * **PVC (PersistentVolumeClaim): 손님의 주문서**
    당신(개발자)은 손님이다. 당신은 "10GB 용량의, 한 번에 하나의 노드에서만 읽고 쓸 수 있는(`ReadWriteOnce`) 스토리지를 원합니다"라는 \*\*요구사항(Claim)\*\*을 담은 주문서(PVC)를 작성하여 쿠버네티스라는 웨이터에게 제출한다. 당신은 이 요리가 주방의 어느 화덕에서, 어떤 요리사에 의해 만들어지는지 전혀 신경 쓰지 않는다.

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: my-app-pvc
    spec:
      accessModes:
        - ReadWriteOnce # 하나의 노드에서만 마운트 가능
      resources:
        requests:
          storage: 10Gi # 10GB의 용량을 요청
    ```

  * **PV (PersistentVolume): 주방에 준비된 실제 요리 재료**
    주방장(인프라 관리자)은 미리 손님을 맞을 준비를 해 둔다. 그는 AWS API를 호출하여 10GB짜리 EBS 볼륨을 미리 만들어두고, "이것은 `ebs-pv-1`이라는 이름의 10GB짜리 `ReadWriteOnce`가 가능한 재료입니다"라는 꼬리표(PV)를 붙여 냉장고에 넣어둔다. 이 PV는 클러스터에 이미 존재하는, 사용할 준비가 된 **실제 스토리지 자원**이다.

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: ebs-pv-1
    spec:
      capacity:
        storage: 10Gi
      accessModes:
        - ReadWriteOnce
      awsElasticBlockStore: # <-- 실제 스토리지의 정체
        volumeID: vol-abcdef123456
    ```

웨이터(쿠버네티스 제어 플레인)는 손님의 주문서(PVC)를 받아서, 냉장고(PV 목록)를 확인하고, 주문서의 요구사항과 정확히 일치하는 재료(PV)를 찾아 둘을 \*\*연결(bind)\*\*해준다. 이제 `my-app-pvc`는 `ebs-pv-1`과 1:1로 묶여 독점적으로 사용된다. 이로써 개발자는 인프라의 구체적인 세부사항을 몰라도 되고, 관리자는 개발자의 애플리케이션 코드를 몰라도 되는 완벽한 분리가 완성된다.

-----

### StorageClass: 주문 즉시 요리하는 자동화된 주방

하지만 만약 손님의 모든 주문에 맞춰 주방장이 매번 수동으로 재료를 준비해야 한다면 얼마나 비효율적인가? 만약 냉장고에 마침 10GB짜리 재료가 없다면, 손님은 하염없이 기다려야만 한다.

바로 이 문제를 해결하는 것이 \*\*스토리지클래스(StorageClass)\*\*다. 스토리지클래스는 미리 만들어진 재료(PV)가 아니라, 특정 종류의 요리를 만드는 **'레시피(Recipe)'** 또는 \*\*'요리 자동화 로봇'\*\*에 해당한다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-fast
provisioner: ebs.csi.aws.com # 어떤 종류의 자동화 로봇을 쓸 것인가
parameters: # 로봇에게 전달할 파라미터
  type: gp3
  fsType: ext4
reclaimPolicy: Delete # PVC가 사라지면 PV와 실제 볼륨도 함께 삭제
```

이제 손님(개발자)은 주문서(PVC)에 특정 레시피의 이름만 적어주면 된다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-fast # <-- "gp3-fast 레시피로 만들어주세요!"
  resources:
    requests:
      storage: 20Gi
```

쿠버네티스는 이 PVC를 보고, `gp3-fast`라는 스토리지클래스에 연결된 자동화 로봇(`ebs.csi.aws.com` 프로비저너)을 호출한다. 그러면 이 로봇이 즉시 AWS API를 호출하여 20GB짜리 `gp3` 타입의 EBS 볼륨을 \*\*동적으로 생성(Dynamically Provision)\*\*하고, 그것을 위한 PV를 만들어 PVC와 자동으로 연결해준다. 더 이상 관리자가 수동으로 PV를 만들 필요가 없는, 완전 자동화된 세상이 열린 것이다. 이것이 바로 \*\*동적 프로비저닝(Dynamic Provisioning)\*\*이다.

[그림: PVC, PV, StorageClass가 상호작용하여 파드에 볼륨을 제공하는 과정]

스테이트풀셋은 바로 이 동적 프로비저닝 메커니즘을 극적으로 활용한다. 스테이트풀셋의 명세 안에 `volumeClaimTemplates`라는 섹션은, 각 파드(`db-0`, `db-1`, ...)가 생성될 때마다 그 파드만을 위한 고유한 PVC(`data-db-0`, `data-db-1`, ...)를 자동으로 생성하기 위한 '주문서 템플릿'이다.

이제 우리는 상태 저장 애플리케이션이 어떻게 안정적인 스토리지와 신원을 보장받는지에 대한 모든 기술적 기반을 이해했다. 하지만 기술이 가능하다고 해서, 그것이 항상 현명한 선택이라는 의미는 아니다. 다음 절에서는 이 기술을 가지고 '데이터베이스를 쿠버네티스 위에서 운영하는 것'의 현실적인 의미와 그 명암을 냉철하게 고찰해볼 것이다.