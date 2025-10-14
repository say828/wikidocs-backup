## KNative Eventing: Kafka, RabbitMQ 이벤트를 KNative 서비스로 연결하기

KNative Serving이 요청에 따라 코드를 실행하는 \*\*'실행 엔진'\*\*이라면, **KNative Eventing**은 이 엔진을 깨우는 **'이벤트 배선(Wiring)'** 시스템입니다. Serving만으로는 HTTP 요청에만 반응할 수 있지만, Eventing을 결합하면 Kafka 메시지, GitHub 웹훅, S3 파일 업로드 등 다양한 이벤트에 반응하는 진정한 이벤트 기반 서버리스 아키텍처를 구축할 수 있습니다.

Eventing의 핵심 목표는 **이벤트 생산자(Producer)와 소비자(Consumer)를 완전히 분리**하는 것입니다.

-----

### KNative Eventing의 핵심 구성 요소

**우체국 시스템 비유 📮:**

  * **Source (소스):** 외부 세계의 이벤트를 KNative 시스템으로 가져오는 **'우편 수집함'**.
  * **Broker (브로커):** 수집된 모든 이벤트를 일단 받아두는 **'중앙 우체국'**.
  * **Trigger (트리거):** 중앙 우체국에 도착한 편지(이벤트)를 보고, 주소(속성)에 따라 적절한 수신자(서비스)에게 편지를 배달하는 **'집배원'**.

#### 1\. Source (소스)

`Source`는 특정 외부 이벤트 소스(예: Kafka 토픽)를 감시하고 있다가, 새로운 이벤트가 발생하면 이를 **CloudEvents**라는 표준화된 포맷으로 변환하여 `Broker`로 전달하는 어댑터입니다. KNative는 `KafkaSource`, `GitHubSource`, `S3Source` 등 다양한 종류의 Source를 기본 제공합니다.

#### 2\. Broker (브로커)

`Broker`는 이벤트의 중앙 허브 역할을 합니다. Source로부터 이벤트를 받아 임시로 저장하고, 어떤 `Trigger`가 이 이벤트를 구독해야 할지 결정합니다.

#### 3\. Trigger (트리거)

`Trigger`는 `Broker`와 최종 목적지인 \*\*구독자(Subscriber)\*\*를 연결하는 규칙입니다. `Trigger`의 가장 강력한 기능은 **이벤트 필터링**입니다. CloudEvent의 속성(Attribute)을 기반으로 "HTTP 헤더의 `type`이 `com.ecommerce.order.paid`인 이벤트만 처리하겠다"와 같이 원하는 이벤트만 선별하여 구독자에게 전달할 수 있습니다.

구독자는 KNative Service(`ksvc`)가 될 수도 있고, 일반적인 쿠버네티스 `Service`가 될 수도 있습니다.

-----

### 실전 예제: Kafka 이벤트를 KNative 서비스로 연결하기

**시나리오:** Kafka의 `greetings` 토픽에 `{"name": "World"}`와 같은 JSON 메시지가 들어오면, 25장 02절에서 만든 `greeter-service`를 트리거하여 "Hello, World"를 로그에 출력하게 만듭니다.

#### 1단계: `KafkaSource` 생성

`greetings` 토픽을 감시하는 `KafkaSource`를 YAML로 정의합니다.

```yaml
# kafka-source.yaml
apiVersion: sources.knative.dev/v1beta1
kind: KafkaSource
metadata:
  name: kafka-greetings-source
spec:
  # 1. 구독할 Kafka 토픽 목록
  topics:
    - greetings
  # 2. Kafka 클러스터의 부트스트랩 서버 주소
  bootstrapServers:
    - kafka.kafka.svc.cluster.local:9092 # 클러스터 내부 Kafka 주소
  # 3. 이벤트가 전달될 목적지 (Sink). 여기서는 'default' 네임스페이스의 기본 Broker
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
```

#### 2단계: `Trigger` 생성

`default` Broker로 들어오는 이벤트 중, `KafkaSource`가 보낸 이벤트만 필터링하여 `greeter-service`로 전달하는 `Trigger`를 정의합니다.

```yaml
# trigger.yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: greeter-service-trigger
spec:
  # 1. 이벤트를 받을 중앙 허브 (Broker)
  broker: default
  # 2. (선택적) 이벤트 필터링 규칙
  #    CloudEvent의 'source' 속성이 kafka-greetings-source인 것만 필터링
  filter:
    attributes:
      source: "https://knative.dev/eventing/kafka/sources/kafka-greetings-source"
  # 3. 이벤트를 최종적으로 전달할 구독자(Subscriber)
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service # KNative Service
      name: greeter-service
```

#### 3단계: 이벤트 발행 및 확인

이제 Kafka 클라이언트를 사용하여 `greetings` 토픽에 메시지를 발행합니다.

```bash
# Kafka Pod에 접속하여 메시지 발행
kafka-console-producer.sh --broker-list kafka:9092 --topic greetings
> {"name": "KNative Eventing"}
```

이 메시지가 발행되는 순간, 다음과 같은 흐름이 자동으로 일어납니다.

1.  `KafkaSource`가 메시지를 감지하여 `Broker`로 전달합니다.
2.  `Trigger`가 이벤트를 필터링하고 `greeter-service`로 전달합니다.
3.  `greeter-service`가 `Scale-to-Zero` 상태였다면, KNative Serving이 즉시 Pod를 1개로 스케일업하여 이벤트를 처리합니다.
4.  `greeter-service`의 로그를 확인하면 "Hello, KNative Eventing"과 유사한 메시지가 출력된 것을 확인할 수 있습니다.

**결론적으로,** KNative Eventing은 다양한 이벤트 소스와 서버리스 함수를 표준화된 방식으로 연결하는 강력한 '접착제' 역할을 합니다. 이를 통해 개발자는 복잡한 이벤트 파이프라인을 선언적인 YAML로 쉽게 구축하고, 특정 클라우드 벤더에 종속되지 않는 진정한 이벤트 기반 서버리스 아키텍처를 쿠버네티스 위에서 실현할 수 있습니다.