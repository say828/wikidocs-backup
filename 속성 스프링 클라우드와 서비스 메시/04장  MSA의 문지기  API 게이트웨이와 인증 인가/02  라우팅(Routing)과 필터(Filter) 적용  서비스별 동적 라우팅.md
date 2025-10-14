## 라우팅(Routing)과 필터(Filter) 적용: 서비스별 동적 라우팅

04장 01절에서 우리는 `application.yml`에 \*\*라우트(Route)\*\*를 정의했습니다. Spring Cloud Gateway에서 라우트는 3가지 핵심 요소로 구성됩니다.

1.  **ID:** 라우트의 고유 식별자 (예: `member-service-route`)
2.  **목적지 URI (Destination URI):** 요청이 최종적으로 전달될 위치 (예: `http://localhost:8081`)
3.  **조건자 (Predicates):** 이 라우트가 적용될 **'조건'**. "만약(IF) 요청 경로가 `/api/v1/members/**`와 일치하면..." (예: `Path=/api/v1/members/**`)

하지만 '조건'만으로는 부족합니다. 조건이 만족되었을 때, 요청을 그냥 전달하기 전에 \*\*'어떤 행동(Action)'\*\*을 추가하고 싶을 수 있습니다. 이 '행동'이 바로 \*\*필터(Filter)\*\*의 역할입니다.

> **"만약(IF) `Predicate` 조건이 참이면, `Filter`를 적용한 후(THEN), 목적지 URI로 요청을 보내라."**

필터는 요청이 마이크로서비스로 전달되기 '전(pre-filter)' 또는 마이크로서비스로부터 응답이 돌아온 '후(post-filter)'에 원하는 로직을 삽입할 수 있는 강력한 메커니즘입니다.

-----

### 1\. 글로벌 필터 (Global Filters): 모든 라우트에 공통 로직 적용하기

모든 API 요청에 대해 공통적으로 '로그(Log)'를 남기고 싶다고 가정해 봅시다. 이 로직을 각 라우트마다 개별적으로 추가하는 것은 비효율적입니다. 이럴 때 **글로벌 필터**를 사용합니다.

글로벌 필터는 `@Component`로 등록하기만 하면, `application.yml`에 정의된 **모든 라우트**에 자동으로 적용됩니다.

#### `GlobalLoggingFilter` 구현

모든 요청의 경로와 상태 코드를 로깅하는 간단한 글로벌 필터를 만들어 봅시다.

```kotlin
// gateway-service/src/main/kotlin/com/ecommerce/gateway/filter/GlobalLoggingFilter.kt
package com.ecommerce.gateway.filter

import org.slf4j.LoggerFactory
import org.springframework.cloud.gateway.filter.GatewayFilterChain
import org.springframework.cloud.gateway.filter.GlobalFilter
import org.springframework.core.Ordered
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import reactor.core.publisher.Mono

@Component
class GlobalLoggingFilter : GlobalFilter, Ordered {

    private val log = LoggerFactory.getLogger(javaClass)

    override fun filter(exchange: ServerWebExchange, chain: GatewayFilterChain): Mono<Void> {
        val request = exchange.request
        
        // --- Pre-Filter: 요청이 마이크로서비스로 가기 전 ---
        log.info(">>>>> [Global Filter] Request: ID={}, URI={}, Path={}", 
            request.id, request.uri, request.path)

        // --- Post-Filter: 마이크로서비스에서 응답이 돌아온 후 ---
        // chain.filter(exchange)가 실행되어야 다음 필터 또는 목적지로 요청이 전달됩니다.
        // .then() 블록은 모든 로직이 끝난 후 실행됩니다.
        return chain.filter(exchange).then(Mono.fromRunnable {
            val response = exchange.response
            log.info("<<<<< [Global Filter] Response: Status={}", response.statusCode)
        })
    }

    // 필터의 실행 순서를 지정합니다. 낮은 값일수록 먼저 실행됩니다.
    override fun getOrder(): Int {
        return 0
    }
}
```

이제 게이트웨이를 재시작하고 아무 API나 호출하면, 위 필터가 자동으로 동작하여 요청과 응답 로그를 콘솔에 출력하는 것을 확인할 수 있습니다.

-----

### 2\. 게이트웨이 필터 (Gateway Filters): 특정 라우트에만 필터 적용하기

이번에는 **특정 라우트**에만 적용되는 필터를 설정해 보겠습니다. 예를 들어, `member-service`로 들어오는 요청에만 특정 헤더를 추가하고 싶을 수 있습니다.

Spring Cloud Gateway는 `AddRequestHeader`, `RewritePath`, `Retry` 등 매우 유용한 내장 필터들을 제공합니다. 이 필터들은 `application.yml`에서 직접 설정할 수 있습니다.

#### `application.yml`에 필터 추가

`member-service-route`에, 게이트웨이를 통과했다는 증표로 `X-Request-Source`라는 요청 헤더를 추가해 보겠습니다.

```yaml
# gateway-service/src/main/resources/application.yml

spring:
  cloud:
    gateway:
      routes:
        - id: member-service-route
          uri: http://localhost:8081
          predicates:
            - Path=/api/v1/members/**
          
          # 1. 'filters' 섹션 추가
          filters:
            # 2. AddRequestHeader 필터 사용
            # Name: 필터 이름, args: 필터에 전달할 인자 (key: value)
            - name: AddRequestHeader
              args:
                name: X-Request-Source # 헤더의 key
                value: gateway-service # 헤더의 value

            # 3. (축약형) AddResponseHeader 필터도 추가해 보자.
            # "필터이름=인자1, 인자2, ..." 형태로도 사용 가능
            - AddResponseHeader=X-Response-Source, gateway-service
```

이제 게이트웨이를 통해 `member-service`를 호출하면, `member-service`는 요청 헤더에 `X-Request-Source: gateway-service`가 추가된 채로 요청을 받게 됩니다. 그리고 클라이언트는 응답 헤더에서 `X-Response-Source: gateway-service`를 확인할 수 있습니다.

-----

이처럼 \*\*조건자(Predicate)\*\*와 \*\*필터(Filter)\*\*의 조합은 Spring Cloud Gateway의 핵심입니다.

  * **Predicate:** "어떤 요청을 이 라우트로 보낼 것인가?" (IF)
  * **Filter:** "그 요청에 어떤 사전/사후 작업을 할 것인가?" (THEN)

우리는 이 강력한 필터 메커니즘을 활용하여, 다음 절에서 MSA의 가장 중요한 공통 관심사인 **'인증 및 인가'** 로직을 게이트웨이 레벨에서 중앙 처리할 것입니다.