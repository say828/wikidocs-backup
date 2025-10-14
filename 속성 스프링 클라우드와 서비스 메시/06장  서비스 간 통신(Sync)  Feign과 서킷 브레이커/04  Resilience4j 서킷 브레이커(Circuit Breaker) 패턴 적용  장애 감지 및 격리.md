## Resilience4j 서킷 브레이커(Circuit Breaker) 패턴 적용: 장애 감지 및 격리

06장 02절의 레스토랑 비유에서, 우리는 "5분 이상 응답이 없으면 포기하라"는 규칙의 필요성을 절감했습니다. **서킷 브레이커(Circuit Breaker)** 패턴은 이 규칙을 시스템적으로 구현한, MSA 안정성의 핵심 패턴입니다.

이름에서 알 수 있듯, 이 패턴은 우리 집의 \*\*'전기 회로 차단기'\*\*에서 영감을 받았습니다.

  * **정상 상태:** 차단기가 **'닫혀(CLOSED)'** 있고, 전기가 정상적으로 흐릅니다. (API 호출이 정상적으로 전달됨)
  * **과부하 발생:** 특정 가전제품(마이크로서비스)에 문제가 생겨 과전류(장애)가 계속 발생합니다.
  * **차단기 작동:** 차단기가 **'열리면서(OPEN)'** 전기 흐름을 끊어버립니다. 이는 문제가 된 가전제품뿐만 아니라, 화재로부터 집 전체(시스템 전체)를 보호하기 위함입니다. (장애가 발생한 서비스로의 API 호출을 즉시 차단함)
  * **상태 확인:** 잠시 후, 차단기는 스스로 **'반만 열린(HALF\_OPEN)'** 상태가 되어 아주 약한 전류를 흘려보내 봅니다. (테스트 요청을 보내 서비스가 복구되었는지 확인)
      * **성공 시:** 문제가 해결되었다고 판단하고, 차단기를 다시 **'닫습니다(CLOSED)'**.
      * **실패 시:** 여전히 문제가 있다고 판단하고, 다시 **'열린(OPEN)'** 상태로 돌아가 대기합니다.

-----

### Resilience4j: 차세대 안정성 라이브러리

과거 Spring Cloud는 `Netflix Hystrix`를 서킷 브레이커 구현체로 사용했지만, 현재는 유지보수가 중단되었습니다. 그 후계자가 바로 **Resilience4j**입니다.

Resilience4j는 서킷 브레이커뿐만 아니라 재시도(Retry), 속도 제한(Rate Limiter), 격벽(Bulkhead) 등 다양한 안정성 패턴을 지원하는 경량화된 함수형 라이브러리입니다. 우리는 이 Resilience4j를 `OpenFeign`과 통합하여, `order-service`가 `product-service`를 호출할 때 발생하는 장애를 지능적으로 감지하고 격리할 것입니다.

-----

### `order-service`에 Resilience4j 서킷 브레이커 적용하기

#### 1\. 의존성 추가

\*\*호출하는 쪽(Client)\*\*인 `order-service`의 `build.gradle.kts`에 Resilience4j와 Spring Cloud Circuit Breaker 스타터 의존성을 추가합니다.

```kotlin
// order-service/build.gradle.kts
dependencies {
    // 1. Spring Cloud Circuit Breaker 스타터 (Resilience4j 구현체 포함)
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j")
    
    // 2. Feign과의 연동을 위해 필요
    implementation("io.github.resilience4j:resilience4j-feign")

    // ... (기존 openfeign, web, data-jpa 등) ...
}
```

#### 2\. `application.yml` 설정

`order-service`의 `application.yml`에 Feign과 서킷 브레이커 동작을 활성화하고, 서킷 브레이커의 상세 규칙을 정의합니다.

```yaml
# order-service/src/main/resources/application.yml

# 1. Feign에 서킷 브레이커 기능 활성화
feign:
  circuitbreaker:
    enabled: true

# 2. Resilience4j 서킷 브레이커 상세 설정
resilience4j:
  circuitbreaker:
    # 3. 설정 그룹(instances) 정의
    instances:
      # 4. 'product-service'라는 이름의 서킷 브레이커 설정
      # 이 이름은 FeignClient의 name/contextId와 일치시킬 수 있음
      product-service:
        # --- 서킷을 OPEN 상태로 전환할 조건 ---
        sliding-window-type: COUNT_BASED # 1. 최근 호출 '횟수' 기반으로 실패율 계산
        sliding-window-size: 10          # 2. 최근 10번의 호출을 샘플링
        failure-rate-threshold: 50       # 3. 그 중 50% (5번) 이상 실패하면 서킷을 OPEN
        
        # --- OPEN 상태에서의 동작 ---
        wait-duration-in-open-state: 5s # 4. OPEN 상태를 5초간 유지. 이후 HALF_OPEN으로 전환

        # --- HALF_OPEN 상태에서의 동작 ---
        permitted-number-of-calls-in-half-open-state: 3 # 5. 3번의 테스트 호출 허용
        
        # --- 기록할 예외와 무시할 예외 ---
        # 기본적으로 모든 예외를 실패로 간주. 특정 예외를 무시할 수도 있음
        # ignore-exceptions:
        #   - com.ecommerce.order.client.ProductNotFoundException
```

  * **`@FeignClient(name = "product-service")`**: Spring Cloud Circuit Breaker는 이 Feign 클라이언트의 `name` 속성을 보고, `resilience4j.circuitbreaker.instances` 설정에서 `product-service`라는 이름의 설정을 찾아 자동으로 적용합니다.

-----

### 서킷 브레이커의 동작 시나리오

이제 `product-service`에 장애가 발생했을 때, 우리의 시스템이 어떻게 동작하는지 살펴봅시다.

1.  **상태: CLOSED (정상)**

      * `product-service`가 정상일 때, `order-service`의 모든 호출은 성공합니다. 서킷 브레이커는 조용히 호출 결과(성공/실패)만 기록합니다.

2.  **장애 발생 및 실패율 증가**

      * `product-service`에 DB 문제로 타임아웃이 발생하기 시작합니다.
      * 최근 10번의 Feign 호출 중 5번이 타임아웃 예외(`feign.RetryableException`)로 실패합니다.
      * 실패율이 50%에 도달하는 순간, Resilience4j는 **서킷을 `CLOSED`에서 `OPEN`으로 전환**합니다\! 💥

3.  **상태: OPEN (호출 차단)**

      * 서킷이 `OPEN` 상태가 되면, `order-service`는 `product-service`로 **실제 네트워크 요청을 보내는 것을 즉시 중단**합니다.
      * 대신, `productServiceClient.getProduct()`를 호출하는 즉시, 네트워크 통신 없이 **`CallNotPermittedException`** 예외를 로컬에서 발생시킵니다.
      * 이는 '그릴 스테이션'으로 가는 길 자체를 막아버려, `order-service`의 스레드가 헛되이 기다리는 것을 원천 차단하고, 장애가 `API Gateway`로 전파되는 것을 막습니다.

4.  **상태: HALF\_OPEN (복구 확인)**

      * `wait-duration-in-open-state`에 설정된 5초가 지나면, 서킷은 `HALF_OPEN` 상태가 됩니다.
      * `permitted-number-of-calls-in-half-open-state`에 설정된 대로, **다음 3번의 `getProduct` 호출**은 실제 `product-service`로 전달됩니다.
          * **만약 3번이 모두 성공하면?** 서킷은 `product-service`가 복구되었다고 판단하고, 상태를 다시 **`CLOSED`로 전환**합니다. 시스템은 정상으로 돌아옵니다.
          * **만약 3번 중 한 번이라도 실패하면?** 서킷은 `product-service`가 여전히 문제라고 판단하고, 상태를 다시 **`OPEN`으로 되돌려** 5초간 대기 상태에 들어갑니다.

이 지능적인 '자동 퓨즈' 덕분에, 우리는 `product-service`의 일시적인 장애가 시스템 전체를 멈추게 하는 '장애 전파'의 공포로부터 벗어났습니다.

하지만 여기서 끝이 아닙니다. 서킷이 `OPEN`되었을 때, 무조건 예외를 던지는 것보다 더 우아한 방법은 없을까요? 다음 절에서는 '폴백(Fallback)'을 통해 실패를 더 부드럽게 처리하는 방법을 배웁니다.