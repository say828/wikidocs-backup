## 게이트웨이의 안정성: 타임아웃, 재시도(Retry) 설정

API 게이트웨이는 MSA의 '단일 진입점'이라는 강력한 장점을 제공하지만, 이는 동시에 \*\*'단일 실패 지점(Single Point of Failure)'\*\*이 될 수도 있다는 의미입니다.

만약 `product-service`가 갑자기 느려지거나 응답하지 않는다면 어떻게 될까요? 아무런 안전장치가 없다면, 게이트웨이는 `product-service`의 응답을 하염없이 기다리게 됩니다. 이윽고 게이트웨이의 모든 스레드(Thread)가 응답 없는 `product-service`를 기다리며 고갈되고, 결국 정상적으로 동작하던 `member-service`나 `order-service`로 가는 요청까지 처리하지 못하는 \*\*'연쇄 장애(Cascading Failure)'\*\*가 발생합니다.

시스템 전체가 마비되는 이 최악의 시나리오를 막기 위해, 우리는 게이트웨이에 반드시 '안정성 패턴'을 적용해야 합니다. 그중 가장 기본적이면서도 필수적인 두 가지가 바로 \*\*타임아웃(Timeout)\*\*과 \*\*재시도(Retry)\*\*입니다.

-----

### 1\. 타임아웃 (Timeout): "빠르게 실패하기(Fail Fast)"

타임아웃은 다운스트림 서비스(예: `product-service`)로부터의 응답을 \*\*'기다리는 최대 시간'\*\*을 설정하는 것입니다.

  * **왜 필요한가?** 응답 없는 서비스를 무한정 기다리는 것은 게이트웨이의 리소스를 낭비하는 최악의 행위입니다. 타임아웃을 설정하면, 약속된 시간이 지났을 때 게이트웨이는 즉시 연결을 끊고 "이 서비스는 현재 응답이 없습니다"라는 실패(예: `504 Gateway Timeout`)를 클라이언트에게 반환합니다.
  * 이는 느린 서비스 하나가 게이트웨이 전체를 마비시키는 것을 막는 핵심적인 **'자원 보호'** 메커니즘입니다. "느리게 실패하는 것"보다 \*\*"빠르게 실패하는 것"\*\*이 전체 시스템 안정성에는 훨씬 이롭습니다.

#### `application.yml` 설정

Spring Cloud Gateway에서는 전체 라우트에 적용되는 글로벌 타임아웃과 특정 라우트에만 적용되는 타임아웃을 설정할 수 있습니다.

```yaml
# gateway-service/src/main/resources/application.yml

spring:
  cloud:
    gateway:
      # --- 1. 글로벌 타임아웃 설정 ---
      httpclient:
        # 전체 라우트에 적용될 응답 타임아웃 (단위: ms)
        # 5초 안에 응답이 오지 않으면 실패 처리
        response-timeout: 5000 
        # 연결 타임아웃 (다운스트림 서비스와 연결을 맺는 시간)
        connect-timeout: 1000

      routes:
        - id: member-service-route
          uri: http://localhost:8081
          predicates:
            - Path=/api/v1/members/**
          # (member-service는 빠르므로 글로벌 타임아웃 5초를 그대로 사용)
        
        - id: some-slow-service-route # 예: 외부 시스템과 연동하는 느린 서비스
          uri: http://localhost:8088
          predicates:
            - Path=/api/v1/slow-service/**
          # --- 2. 특정 라우트에 대한 타임아웃 재정의 ---
          metadata:
            response-timeout: 10000 # 이 라우트는 10초까지 기다림
            connect-timeout: 2000
```

-----

### 2\. 재시도 (Retry): "일시적인 장애 극복하기"

마이크로서비스는 배포나 일시적인 네트워크 문제로 아주 잠깐(수십\~수백 ms) 응답하지 못하는 경우가 빈번하게 발생합니다. 이런 **'일시적인(Transient)' 장애** 때문에 클라이언트가 바로 에러를 받는 것은 좋지 않은 사용자 경험입니다.

\*\*재시도(Retry)\*\*는 이런 일시적인 오류가 발생했을 때, 게이트웨이가 자동으로 요청을 몇 번 더 보내보는 기능입니다.

  * **왜 필요한가?** 첫 번째 시도는 실패했지만, 100ms 뒤 두 번째 시도는 성공할 수 있습니다. 재시도를 통해 시스템은 사소한 장애를 스스로 '회복'하고 안정성을 높일 수 있습니다.

#### `application.yml` 설정 (Retry 필터)

재시도는 Spring Cloud Gateway의 내장 `Retry` 필터를 사용하여 라우트별로 설정합니다.

```yaml
# gateway-service/src/main/resources/application.yml

spring:
  cloud:
    gateway:
      routes:
        - id: product-service-route # 상품 서비스는 재시도 정책 적용
          uri: http://localhost:8082
          predicates:
            - Path=/api/v1/products/**
          filters:
            - name: Retry
              args:
                retries: 3 # 1. 최대 3번까지 재시도
                statuses: # 2. 어떤 HTTP 상태 코드일 때 재시도할 것인가?
                  - BAD_GATEWAY # 502
                  - SERVICE_UNAVAILABLE # 503
                  - GATEWAY_TIMEOUT # 504
                methods: # 3. 어떤 HTTP 메서드에만 재시도할 것인가?
                  - GET # 멱등성(Idempotent)이 보장되는 GET, HEAD, OPTIONS 등에만 적용
                backoff: # 4. 재시도 간격 설정 (Exponential Backoff)
                  firstBackoff: 100ms # 첫 시도까지 100ms 대기
                  maxBackoff: 500ms # 최대 500ms까지 늘어남
                  factor: 2 # 대기 시간을 2배씩 증가 (100ms -> 200ms -> 400ms)
                  basedOnPreviousValue: true
```

  * **⛔️ 매우 중요:** 재시도는 `GET`처럼 여러 번 호출해도 결과가 같은 \*\*'멱등성(Idempotency)'\*\*이 보장되는 HTTP 메서드에만 적용해야 합니다. 만약 `POST` (생성) 요청을 재시도하면, **'중복 주문'이나 '중복 결제'** 같은 심각한 문제를 일으킬 수 있습니다.

-----

**결론적으로**, 타임아웃으로 '최악의 상황'을 방지하여 시스템 전체를 보호하고, 재시도로 '사소한 장애'를 극복하여 안정성을 높였습니다. 이 두 가지 설정은 프로덕션 레벨의 API 게이트웨이가 갖추어야 할 최소한의, 그러나 가장 중요한 안정성 장치입니다.

물론, 이것만으로는 부족합니다. 더 지능적으로 장애를 감지하고 격리하는 **'서킷 브레이커(Circuit Breaker)'** 패턴은 06장에서 더 깊이 다룰 것입니다.