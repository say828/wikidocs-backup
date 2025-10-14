## 이커머스 '이미지 리사이징' 기능을 KNative 서비스로 구축하기

KNative Serving과 Eventing의 이론을 모두 학습했으니, 이제 이들을 조합하여 실제 이커머스 시스템에 필요한 **'이미지 리사이징'** 기능을 완전한 서버리스 워크플로우로 구축해 보겠습니다.

이 기능은 서버리스의 장점을 극대화할 수 있는 완벽한 사용 사례입니다. 이미지는 매일 꾸준히 업로드되는 것이 아니라, 판매자가 상품을 등록하는 특정 시점에만 비정기적으로 업로드되기 때문입니다.

-----

### 워크플로우 설계

1.  **[트리거]** 판매자가 상품의 고해상도 원본 이미지를 **S3와 같은 오브젝트 스토리지**의 `original-images` 버킷에 업로드합니다.
2.  **[이벤트 감지]** KNative Eventing의 \*\*`S3Source`\*\*가 이 '파일 업로드' 이벤트를 감지하여, 표준 CloudEvent로 변환한 뒤 \*\*`Broker`\*\*로 보냅니다.
3.  **[이벤트 라우팅]** \*\*`Trigger`\*\*가 이 이벤트를 받아, 우리의 이미지 리사이징 서비스인 \*\*`image-resizer` (KNative Service)\*\*로 전달합니다.
4.  **[실행]** **KNative Serving**은 `image-resizer` 서비스가 0개의 Pod 상태였다면 즉시 1개로 스케일업하여 이벤트를 처리하게 합니다.
5.  **[비즈니스 로직]** `image-resizer` 서비스는 이벤트 정보를 바탕으로 `original-images` 버킷에서 원본 이미지를 다운로드합니다.
6.  이미지를 썸네일, 중간 사이즈, 대형 사이즈 등 여러 규격으로 리사이징합니다.
7.  결과물 이미지들을 `resized-images` 버킷에 업로드합니다.
8.  **[Scale-to-Zero]** 작업 완료 후, 일정 시간 동안 더 이상 이미지 업로드 이벤트가 없으면, `image-resizer` 서비스의 Pod는 다시 0개로 축소됩니다.

-----

### 1단계: 이미지 리사이징 애플리케이션 (컨테이너)

먼저, 실제 이미지 리사이징 로직을 수행할 간단한 웹 애플리케이션을 작성하고 컨테이너 이미지(`ecommerce/image-resizer:0.0.1`)로 빌드합니다. 이 애플리케이션은 KNative Eventing이 보내주는 **CloudEvent** 포맷의 HTTP `POST` 요청을 받을 수 있는 엔드포인트를 가져야 합니다.

애플리케이션은 CloudEvent 페이로드에서 `bucket` 이름과 `object key`(파일 이름)를 파싱한 뒤, AWS S3 SDK 같은 라이브러리를 사용하여 파일을 다운로드/리사이징/업로드하는 로직을 수행합니다.

-----

### 2단계: KNative 리소스 YAML 작성

이제 이 컨테이너를 서버리스 워크플로우에 통합하기 위한 KNative 리소스들을 정의합니다.

#### KNative Service (ksvc) 정의

`Scale-to-Zero`로 동작할 `image-resizer` 서비스를 배포합니다.

```yaml
# knative-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: image-resizer
spec:
  template:
    spec:
      containers:
        - image: ecommerce/image-resizer:0.0.1
          env:
            - name: TARGET_BUCKET
              value: "resized-images"
```

#### Event Source (S3Source) 정의

`original-images` 버킷을 감시하는 `S3Source`를 생성합니다.

```yaml
# knative-source.yaml
apiVersion: sources.knative.dev/v1alpha1 # (API 버전은 소스 종류에 따라 다를 수 있음)
kind: S3Source
metadata:
  name: s3-original-images-source
spec:
  # 1. 감시할 S3 버킷 정보
  bucket: original-images
  # 2. 어떤 이벤트를 감지할지 (예: 파일 생성)
  eventTypes:
    - "s3:ObjectCreated:*"
  # 3. 이벤트를 보낼 목적지 (기본 Broker)
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
```

#### Trigger 정의

`Broker`에 도착한 이벤트를 `image-resizer` 서비스로 연결합니다.

```yaml
# knative-trigger.yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: image-resizer-trigger
spec:
  broker: default
  # 1. (선택적) S3 소스로부터 온 이벤트만 필터링
  filter:
    attributes:
      source: "https://knative.dev/eventing/awssqs/sources/..." 
  # 2. 이벤트의 최종 구독자 지정
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: image-resizer
```

-----

### 최종 결과

위 YAML 파일들을 `kubectl apply` 명령어로 모두 클러스터에 배포하고, `original-images` S3 버킷에 이미지 파일을 업로드하면, 설계했던 워크플로우가 자동으로 실행되는 것을 확인할 수 있습니다. 평소에는 `image-resizer` Pod가 보이지 않다가, 파일이 업로드되는 순간 Pod가 생성되어 로그를 남기고, 작업이 끝난 뒤 잠시 후 다시 사라지는 완전한 서버리스 동작을 관찰할 수 있습니다.

이것으로 25장을 마칩니다. 우리는 KNative Serving과 Eventing을 결합하여, **특정 클라우드 제공업체에 종속되지 않고**, 우리가 직접 통제하는 쿠버네티스 클러스터 위에서 정교하고, 비용 효율적이며, 탄력적인 서버리스 워크플로우를 성공적으로 구축했습니다.

이제 이 책의 마지막 여정으로, 지금까지 배운 모든 기술을 바탕으로 미래의 백엔드 아키텍트가 나아가야 할 방향과 그 역할의 변화에 대해 고찰해 보겠습니다.