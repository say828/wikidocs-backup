# 09장: 세상과 클러스터를 연결하는 관문, Ingress와 Gateway

우리는 8장에서 클러스터라는 제국의 내부 교통망과 국경 방어 체계를 완벽하게 구축했다. 파드들은 CNI를 통해 신원을 얻고, DNS를 통해 서로를 발견하며, 네트워크 정책이라는 견고한 성벽 안에서 허가된 경로로만 안전하게 통신한다. 이로써 우리는 고도로 안정적이고 보안이 강화된 내부 시스템을 완성했다. 하지만 이 위대한 제국이 외부 세계와 교역하지 않고 고립된다면, 그 존재 가치는 퇴색하고 말 것이다. 🏰

결국 우리의 애플리케이션은 최종 사용자인 외부의 클라이언트들을 만나야만 한다. 어떻게 하면 인터넷이라는 거친 바다를 항해하는 수많은 요청들을 우리 클러스터 안의 정확한 서비스로 안전하고 효율적으로 안내할 수 있을까? `Service` 리소스의 `type: LoadBalancer`는 각 서비스마다 값비싼 클라우드 로드밸런서를 하나씩 할당하는, 가장 단순하지만 비효율적인 해결책일 뿐이다.

이 장에서는 클러스터의 '관문(Gateway)' 역할을 수행하는 L7 라우팅의 세계로 깊이 들어간다. 우리는 쿠버네티스의 초기 표준이었던 **Ingress**의 아키텍처와 그 진화 과정을 살펴보고, 오늘날 생태계를 지배하는 대표적인 Ingress Controller들—**NGINX, Traefik, Contour**—의 트레이드오프를 날카롭게 비교 분석할 것이다. 또한, 모든 현대 웹 서비스의 기본 소양인 HTTPS를 **Cert-manager**를 통해 어떻게 자동화하는지 알아보고, 마지막으로 Ingress의 근본적인 한계를 극복하기 위해 태어난 차세대 표준, **Gateway API**가 열어갈 미래를 조망할 것이다.

-----

## 00\. Ingress Controller 아키텍처: 트래픽 라우팅의 진화 과정

클러스터 외부의 트래픽을 내부 서비스로 연결하는 가장 원시적인 방법은 `Service` 리소스의 타입을 `LoadBalancer`로 설정하는 것이다. 이렇게 하면 클라우드 제공업체는 해당 서비스를 위해 외부에서 접근 가능한 공인 IP 주소를 가진 L4 로드밸런서를 자동으로 프로비저닝해준다. 이는 작동은 하지만, 다음과 같은 심각한 문제를 안고 있다.

  * **비용 문제:** 서비스 하나당 하나의 로드밸런서가 생성된다. 만약 당신이 100개의 마이크로서비스를 외부에 노출해야 한다면, 100개의 값비싼 로드밸런서 비용을 감당해야 한다.
  * **기능의 한계:** L4 로드밸런서는 IP 주소와 포트만을 이해한다. `/api/users` 요청은 A 서비스로, `/api/orders` 요청은 B 서비스로 보내는 것과 같은 HTTP 경로 기반 라우팅이나, `a.example.com`과 `b.example.com`을 구분하는 호스트 기반 라우팅과 같은 L7 수준의 지능적인 트래픽 제어가 불가능하다.

이 문제를 해결하기 위해 쿠버네티스는 **Ingress**라는 L7 라우팅을 위한 표준 API 리소스를 도입했다. Ingress의 핵심 아이디어는, 단 하나의 공인 IP 주소를 가진 로드밸런서를 클러스터의 \*\*'공동 현관'\*\*으로 사용하고, 이 공동 현관에 들어온 모든 HTTP/HTTPS 요청을 규칙에 따라 적절한 내부 서비스(각각의 집)로 분배하는 것이다.

Ingress 시스템은 두 개의 분리된 컴포넌트로 구성되며, 이 둘의 관계를 이해하는 것이 핵심이다.

1.  **Ingress 리소스 (The Rule):** 이것은 당신이 작성하는 YAML 파일이다. 여기에는 "만약 `foo.bar.com/path`로 요청이 들어오면, `foo-service`의 80번 포트로 보내라"와 같은 \*\*라우팅 규칙(Routing Rule)\*\*만이 선언적으로 기술된다. 이 리소스 자체는 아무런 네트워킹 능력도 없다. 그것은 그저 '의도'를 담은 종이 위의 설계도일 뿐이다.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: my-app-ingress
    spec:
      rules:
      - host: "myapp.example.com"
        http:
          paths:
          - path: /user
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /order
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
    ```

2.  **Ingress Controller (The Engine):** 이것이야말로 실제 트래픽을 처리하는 **엔진**이다. Ingress Controller는 클러스터 안에서 실행되는 파드(보통 디플로이먼트)이며, 그 본질은 NGINX, Envoy, Traefik과 같은 강력한 리버스 프록시(Reverse Proxy) 소프트웨어다. 이 컨트롤러는 쿠버네티스 API 서버를 계속 감시하다가, 새로운 Ingress 리소스가 생성되거나 변경되면, 그 규칙을 읽어들여 자신의 리버스 프록시 설정(예: `nginx.conf`)을 동적으로 업데이트한다.

이 두 컴포넌트는 다음과 같은 흐름으로 함께 동작한다.

  * **1. 외부 트래픽 유입:** 사용자의 요청은 먼저 DNS를 통해 Ingress Controller가 노출된 공인 IP 주소로 향한다. 이 공인 IP는 보통 Ingress Controller 파드 자체를 노출시키는 `Service` (type: LoadBalancer)에 의해 제공된다.
  * **2. Ingress Controller 수신:** 트래픽은 이 공동 현관, 즉 Ingress Controller 파드에 도달한다.
  * **3. 규칙 기반 라우팅:** Ingress Controller는 요청의 호스트 헤더(`myapp.example.com`)와 경로(`/user`)를 확인하고, 자신이 메모리에 로드해둔 Ingress 규칙에 따라 이 요청을 `user-service`로 보내야 함을 결정한다.
  * **4. 내부 서비스로 전달:** Ingress Controller는 이 요청을 `user-service`의 ClusterIP로 전달하고, 최종적으로 `user-service`가 선택하고 있는 애플리케이션 파드에 도달하게 된다.

이 아키텍처 덕분에 우리는 단 하나의 외부 로드밸런서와 Ingress Controller를 통해 수십, 수백 개의 내부 서비스를 비용 효율적이고 지능적으로 외부에 노출할 수 있게 되었다.

하지만 쿠버네티스의 Ingress API 명세 자체는 매우 최소한의 기능(호스트, 경로, TLS)만을 표준으로 정의했다. 재시도(retry), 타임아웃, 트래픽 가중치 분배와 같은 더 복잡하고 중요한 기능들은 각 Ingress Controller 구현체들이 \*\*어노테이션(Annotation)\*\*이라는 비표준적인 방식으로 각자 구현하기 시작했다. 이는 특정 Ingress Controller에 대한 종속성을 심화시키고, 이식성을 해치는 원인이 되었다.

이러한 한계와 더불어, 플랫폼 관리자와 애플리케이션 개발자 간의 역할 분담이 모호하다는 문제점 때문에, 커뮤니티는 Ingress를 넘어설 새로운 표준을 모색하기 시작했다. 이것이 바로 이 장의 마지막에 다룰 Gateway API의 등장 배경이다. 하지만 그전에, 현재 Ingress Controller 시장을 삼분하고 있는 거인들의 특징을 비교하며 우리에게 맞는 엔진을 선택하는 방법을 알아보자.