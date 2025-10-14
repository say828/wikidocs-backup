# 05장: MSA의 등대: 서비스 디스커버리와 중앙 설정

## 00\. 서비스가 서로를 찾는 방법: 서비스 디스커버리(Service Discovery)의 원리

04장에서 우리는 API 게이트웨이의 `application.yml`에 마이크로서비스의 주소를 직접 하드코딩했습니다.

```yaml
# gateway-service/application.yml (나쁜 예시 👎)
spring:
  cloud:
    gateway:
      routes:
        - id: member-service-route
          uri: http://localhost:8081 # 1. 주소 하드코딩
        - id: product-service-route
          uri: http://localhost:8082 # 2. 주소 하드코딩
```

이 방식은 로컬 개발 환경에서는 동작하지만, 실제 프로덕션 환경에서는 **절대 사용 불가능한** 방법입니다. 왜일까요?

클라우드 네이티브 환경(예: 쿠버네티스, AWS EC2)에서 마이크로서비스는 '동적(Dynamic)'으로 움직입니다.

1.  **동적인 IP와 포트:** 서비스는 서버 장애나 배포 시 언제든지 새로운 IP 주소와 임의의(Random) 포트를 할당받아 재시작될 수 있습니다. 어제의 `product-service`는 `10.0.1.5:8082`였지만, 오늘의 `product-service`는 `10.0.2.8:49152`일 수 있습니다.
2.  **동적인 개수 (Scale-out):** 트래픽이 몰리면 `product-service`는 1대에서 10대로 자동 확장(Auto-scaling)될 수 있습니다. 이 10대의 서비스는 모두 다른 IP 주소를 가집니다. 게이트웨이는 이 10개의 주소를 모두 알고, 요청을 골고루 분산(Load Balancing)해야 합니다.

이처럼 끊임없이 변하는 MSA 환경에서, 하드코딩된 주소는 쓸모가 없습니다. `order-service`는 `product-service`의 현재 실제 IP 주소가 무엇인지, 그리고 몇 대가 떠 있는지 어떻게 알 수 있을까요?

-----

### 해답: 서비스 디스커버리, "MSA의 전화번호부"

이 문제를 해결하는 것이 바로 **서비스 디스커버리(Service Discovery)** 패턴입니다. 이는 마치 '전화번호부' 또는 'DNS'와 같습니다.

1.  **서비스 등록 (Service Registration):**

      * 모든 마이크로서비스(`member-service`, `product-service` 등)는 시작(Bootstrap)될 때, 자신의 \*\*'서비스 이름'\*\*과 **'실제 IP 주소:포트'** 정보를 중앙의 **'서비스 레지스트리(Service Registry)'** 서버에 등록합니다.
      * "안녕, 레지스트리 서버? 나는 `product-service`이고, 내 현재 주소는 `10.0.1.5:8082`야."
      * 서비스는 주기적으로 "나 아직 살아있어"라는 신호(Heartbeat)를 레지스트리에 보냅니다.

2.  **서비스 발견 (Service Discovery):**

      * API 게이트웨이나 `order-service`가 `product-service`를 호출해야 할 때, 더 이상 IP 주소를 직접 찾지 않습니다.
      * 대신, 레지스트리 서버에 \*\*'서비스 이름'\*\*을 물어봅니다.
      * "안녕, 레지스트리 서버? `product-service`라는 이름으로 등록된 서버들 주소 목록 좀 알려줄래?"

3.  **로드 밸런싱 및 호출:**

      * 레지스트리 서버는 현재 살아있는 `product-service`들의 IP 주소 목록(예: `[10.0.1.5:8082, 10.0.2.8:49152, ...]`)을 반환합니다.
      * 게이트웨이는 이 목록 중 하나를 선택하여(보통 Round-robin 방식의 클라이언트 사이드 로드 밸런싱) 실제 요청을 보냅니다.

-----

이 '서비스 레지스트리' 덕분에, 각 서비스는 서로의 물리적인 주소에 대한 의존성에서 완벽하게 해방됩니다. 오직 논리적인 \*\*'서비스 이름'\*\*만 알면 통신이 가능해집니다.

이제 게이트웨이의 `application.yml`은 다음과 같이 바뀔 수 있습니다.

```yaml
# gateway-service/application.yml (좋은 예시 👍)
spring:
  cloud:
    gateway:
      routes:
        - id: member-service-route
          # 'lb'는 Load Balancer를 의미
          # "서비스 레지스트리에서 MEMBER-SERVICE라는 이름을 찾아 로드 밸런싱 해줘"
          uri: lb://MEMBER-SERVICE
        - id: product-service-route
          uri: lb://PRODUCT-SERVICE
```

이 장에서는 스프링 클라우드 생태계의 대표적인 서비스 레지스트리 구현체인 `Netflix Eureka`를 통해 이 'MSA의 등대'를 구축하는 방법을 배울 것입니다. (주: 현재는 Consul이나 쿠버네티스 네이티브 방식이 더 선호되지만, Eureka는 서비스 디스커버리의 기본 원리를 이해하는 데 가장 훌륭한 교육 도구입니다.)