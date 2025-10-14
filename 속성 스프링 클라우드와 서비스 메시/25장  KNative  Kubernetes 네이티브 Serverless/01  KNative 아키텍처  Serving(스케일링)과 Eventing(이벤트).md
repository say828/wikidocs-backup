## KNative 아키텍처: Serving(스케일링)과 Eventing(이벤트)

KNative는 단일 컴포넌트가 아니라, 서버리스 애플리케이션의 라이프사이클을 관리하는 두 개의 핵심 컴포넌트, **Serving**과 **Eventing**으로 구성됩니다. 이 둘은 서로를 보완하며 완전한 서버리스 플랫폼을 이룹니다.

---

### 1. KNative Serving: 요청 기반 컴퓨팅과 Scale-to-Zero의 엔진 🚀

**KNative Serving**은 서버리스 워크로드의 **'컴퓨팅'** 부분을 책임집니다. Serving의 핵심 목표는 개발자가 배포한 컨테이너를 **HTTP 요청에 따라 자동으로 스케일링**하고, 트래픽이 없을 때는 **Pod의 개수를 0으로 줄이는(Scale-to-Zero)** 것입니다.

개발자는 더 이상 `Deployment`, `Service`, `HPA`, `Istio VirtualService` 등 복잡한 쿠버네티스 리소스를 직접 다룰 필요가 없습니다. KNative Serving이 제공하는 **`Service`**라는 단일 리소스 하나만 정의하면, Serving이 내부적으로 이 모든 복잡한 리소스들을 **자동으로 생성하고 관리**해 줍니다.

#### KNative Serving의 핵심 리소스 (CRD)

* **Service (`ksvc`):** 개발자가 직접 생성하는 **최상위 리소스**입니다. 이 리소스 하나를 정의하면, 아래의 `Route`, `Configuration`, `Revision`이 자동으로 생성되고 관리됩니다.
* **Configuration:** 배포의 '원하는 상태'를 정의합니다. 어떤 컨테이너 이미지를 사용할지, 어떤 환경 변수를 설정할지 등을 담고 있습니다. `Configuration`이 업데이트될 때마다 새로운 `Revision`이 생성됩니다.
* **Revision:** **'코드와 설정의 불변 스냅샷(Immutable Snapshot)'**입니다. `Configuration`이 변경될 때마다 생성되는, 특정 시점의 버전입니다. 예를 들어, 컨테이너 이미지를 `v1`에서 `v2`로 바꾸면, 새로운 `Revision`(`v2`)이 생성되고 기존 `Revision`(`v1`)도 그대로 남아있습니다. 이 불변성 덕분에 안정적인 롤백과 버전별 트래픽 분배가 가능해집니다.
* **Route:** 특정 `Revision`(들)으로 트래픽을 라우팅하는 네트워크 엔드포인트(URL)를 관리합니다. 내부적으로 Istio의 `VirtualService`를 제어하여, `Revision` 간의 트래픽을 정교하게 분배(예: `v1`에 90%, `v2`에 10%)하는 역할을 합니다.

**요약:** 개발자는 `KNative Service`에 "이 컨테이너 이미지로 앱을 실행해 줘"라고 선언하기만 하면, KNative Serving이 알아서 이 앱을 외부에 노출하고, 트래픽에 따라 Pod를 0개에서 N개까지 자동으로 스케일링하는 모든 복잡한 작업을 처리합니다.

---

### 2. KNative Eventing: 이벤트 흐름을 위한 범용 허브 HUB

**KNative Eventing**은 서버리스 워크로드의 **'트리거'** 부분을 책임집니다. Eventing의 핵심 목표는 다양한 **이벤트 소스(Event Source)**로부터의 이벤트를 **표준화된 방식**으로 수신하고, 이를 적절한 **구독자(Subscriber)**에게 안정적으로 전달하는 것입니다.

Eventing은 이벤트 생산자와 소비자를 완벽하게 분리하여, 느슨하게 결합된 이벤트 기반 아키텍처를 손쉽게 구축할 수 있도록 돕습니다.



#### KNative Eventing의 핵심 리소스 (CRD)

* **Source:** 외부 시스템(예: Kafka 토픽, GitHub 웹훅, AWS S3 버킷)에서 발생하는 이벤트를 KNative 이벤트 시스템 **내부로 가져오는** '어댑터'입니다. 모든 이벤트는 **CloudEvents**라는 표준 포맷으로 변환됩니다.
* **Broker:** 이벤트의 **'중앙 허브' 또는 '우체통'** 역할을 합니다. Source는 이벤트를 특정 Broker로 보냅니다.
* **Trigger:** **'Broker'와 '구독자'를 연결**하는 규칙입니다. Trigger는 특정 속성(Attribute)을 가진 이벤트만 필터링하여, 지정된 구독자(예: KNative Service)에게 전달하는 역할을 합니다.

#### Serving과 Eventing의 협업

1.  외부에서 이벤트가 발생합니다. (예: Kafka 토픽에 메시지 도착)
2.  **`KafkaSource`**(Eventing)가 이 이벤트를 감지하여 CloudEvent로 변환한 뒤, **`Broker`**(Eventing)로 보냅니다.
3.  **`Trigger`**(Eventing)가 이 이벤트를 필터링하고, 구독자로 지정된 **`KNative Service`**(Serving)로 이벤트를 전달합니다.
4.  이 이벤트(HTTP POST 요청)를 받은 **`KNative Serving`**은, 만약 해당 서비스가 `Scale-to-Zero` 상태였다면, 즉시 Pod를 1개로 스케일업하여 요청을 처리합니다.

**결론적으로,** `KNative Serving`은 요청에 따라 코드를 실행하고 스케일링하는 **'실행 엔진'**의 역할을, `KNative Eventing`은 이 실행 엔진을 깨우는 **'이벤트 배선(Wiring)'**의 역할을 담당합니다. 이 두 가지가 결합하여, 개발자는 어떤 쿠버네티스 환경에서든 이식성 높은 완전한 서버리스 애플리케이션을 구축할 수 있게 됩니다.