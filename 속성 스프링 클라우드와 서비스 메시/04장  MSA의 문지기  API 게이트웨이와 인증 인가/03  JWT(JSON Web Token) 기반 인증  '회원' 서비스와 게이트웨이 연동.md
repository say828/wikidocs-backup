## JWT(JSON Web Token) 기반 인증: '회원' 서비스와 게이트웨이 연동

이제 우리는 MSA의 '문지기'인 API 게이트웨이에 '보안 요원'을 배치하여, 아무나 우리 시스템에 들어올 수 없도록 만들 것입니다. 즉, **인증(Authentication)** 로직을 구현합니다.

04장 00절에서 논의했듯이, 모든 마이크로서비스가 개별적으로 '인증'을 처리하는 것은 엄청난 중복과 비효율을 낳습니다. 따라서 우리는 이 '인증'이라는 \*\*횡단 관심사(Cross-Cutting Concern)\*\*를 **API 게이트웨이**에서 중앙 집중적으로 처리할 것입니다.

이를 위한 가장 표준적인 기술이 바로 \*\*JWT(JSON Web Token)\*\*입니다.

-----

### 왜 JWT인가? 상태 없는(Stateless) 인증의 힘

전통적인 세션(Session) 기반 인증은 서버가 사용자의 로그인 상태를 메모리나 DB에 저장해야 합니다. 하지만 MSA 환경에서는 `member-service`에서 생성된 세션 정보를 `product-service`가 알 수 없습니다.

JWT는 이 문제를 해결하는 **'상태 없는(Stateless)'** 인증 방식입니다. 서버는 사용자의 정보를 '토큰'이라는 암호화된 문자열로 만들어 클라이언트에게 전달하고, 서버 자신은 아무것도 저장하지 않습니다.

  * **JWT 구조:** JWT는 `.`으로 구분된 세 부분으로 구성됩니다: `Header.Payload.Signature`
      * **Header:** 토큰의 타입(JWT)과 서명 알고리즘(예: HS256)을 명시합니다.
      * **Payload:** 사용자의 정보(Claim)를 담습니다. 우리는 여기에 `회원 ID (memberId)`와 만료 시간(exp) 등을 담을 것입니다.
      * **Signature:** `Header`와 `Payload`가 위변조되지 않았음을 증명하는 '전자 서명'입니다. 서버만이 알고 있는 \*\*'비밀 키(Secret Key)'\*\*로 생성됩니다.

게이트웨이는 클라이언트가 보낸 JWT의 '서명'을 자신의 '비밀 키'로 검증함으로써, 이 토큰이 위변조되지 않은 유효한 토큰임을 신뢰할 수 있습니다.

-----

### 인증 흐름 설계

1.  **로그인 및 토큰 발급 (by `member-service`):**

      * 사용자는 `POST /api/v1/members/login` API를 호출합니다. 이 요청은 '인증'이 필요 없으므로 게이트웨이를 통과하여 `member-service`로 전달됩니다.
      * `member-service`는 이메일/패스워드를 검증하고, 성공 시 **`회원 ID`를 담은 JWT**를 생성하여 클라이언트에게 응답합니다.

2.  **인증이 필요한 API 호출 (by Client):**

      * 클라이언트는 발급받은 JWT를 `Authorization` 헤더에 `Bearer [JWT]` 형태로 담아 `GET /api/v1/orders/my-orders` 같은 API를 호출합니다.

3.  **게이트웨이에서의 토큰 검증 (by `gateway-service`):**

      * API 게이트웨이는 이 요청을 받습니다.
      * 게이트웨이는 `Authorization` 헤더에서 JWT를 추출합니다.
      * 자신이 가진 '비밀 키'로 JWT의 서명을 검증합니다.
      * **검증 성공 시:** JWT의 Payload에서 `회원 ID`를 추출한 뒤, **`X-Member-Id: 123`** 과 같은 새로운 헤더를 추가하여 실제 `order-service`로 요청을 전달합니다.
      * **검증 실패 시 (위변조, 만료 등):** `401 Unauthorized` 에러를 클라이언트에게 즉시 반환하고, 내부 서비스로는 요청을 보내지 않습니다.

4.  **내부 서비스의 비즈니스 로직 처리 (by `order-service`):**

      * `order-service`는 JWT의 존재 자체를 알 필요가 없습니다. 그저 게이트웨이가 추가해 준 **`X-Member-Id` 헤더를 100% 신뢰**하고, "아, 123번 회원의 주문 내역을 조회하는 요청이구나"라고 인지한 뒤 비즈니스 로직을 처리합니다.

-----

### 1\. `gateway-service`에 JWT 의존성 추가 및 유틸리티 작성

먼저 `gateway-service`의 `build.gradle.kts`에 JWT 라이브러리(`jjwt`)를 추가합니다.

```kotlin
// gateway-service/build.gradle.kts
dependencies {
    // ... 기존 의존성 ...
    
    // JWT 라이브러리
    implementation("io.jsonwebtoken:jjwt-api:0.12.5")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.5")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.5")
}
```

다음으로, 토큰 검증 로직을 담을 `JwtUtil` 클래스와 `application.yml`에 비밀 키를 추가합니다.

```yaml
# gateway-service/src/main/resources/application.yml
jwt:
  secret: 'a_very_long_and_secure_secret_key_for_ecommerce_project_that_is_at_least_256_bits_long' # ⛔️ 실제 프로덕션에서는 환경 변수 등으로 관리해야 합니다.
```

```kotlin
// gateway-service/src/main/kotlin/com/ecommerce/gateway/util/JwtUtil.kt
package com.ecommerce.gateway.util

import io.jsonwebtoken.Claims
import io.jsonwebtoken.Jwts
import io.jsonwebtoken.security.Keys
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Component
import java.nio.charset.StandardCharsets
import javax.crypto.SecretKey

@Component
class JwtUtil(@Value("\${jwt.secret}") secret: String) {

    // 1. application.yml의 secret 값을 HS256 알고리즘에 맞는 SecretKey 객체로 변환
    private val secretKey: SecretKey = Keys.hmacShaKeyFor(secret.toByteArray(StandardCharsets.UTF_8))

    /**
     * 토큰에서 모든 Claim(정보) 추출
     */
    fun getAllClaims(token: String): Claims {
        return Jwts.parser()
            .verifyWith(secretKey)
            .build()
            .parseSignedClaims(token)
            .payload
    }

    /**
     * 토큰에서 회원 ID(subject) 추출
     */
    fun getMemberId(token: String): Long {
        return getAllClaims(token).subject.toLong()
    }
    
    /**
     * 토큰 만료 여부 확인
     */
    fun isTokenExpired(token: String): Boolean {
        return getAllClaims(token).expiration.before(Date())
    }
}
```

-----

### 2\. 커스텀 인증 필터 (AuthenticationFilter) 구현

이제 이 `JwtUtil`을 사용하여 실제 인증 로직을 수행할 커스텀 필터를 만듭니다.

```kotlin
// gateway-service/src/main/kotlin/com/ecommerce/gateway/filter/AuthenticationFilter.kt
package com.ecommerce.gateway.filter

import com.ecommerce.gateway.util.JwtUtil
import io.jsonwebtoken.JwtException
import org.springframework.cloud.gateway.filter.GatewayFilter
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory
import org.springframework.http.HttpHeaders
import org.springframework.http.HttpStatus
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange

@Component
class AuthenticationFilter(
    private val jwtUtil: JwtUtil
) : AbstractGatewayFilterFactory<AuthenticationFilter.Config>(Config::class.java) {

    // 설정 클래스 (현재는 사용하지 않음)
    class Config
    
    override fun apply(config: Config): GatewayFilter {
        return GatewayFilter { exchange, chain ->
            val request = exchange.request
            
            // 1. 공개된 엔드포인트는 필터링하지 않음 (예: /join, /login)
            if (request.uri.path.contains("/join") || request.uri.path.contains("/login")) {
                return@GatewayFilter chain.filter(exchange)
            }
            
            // 2. 'Authorization' 헤더 존재 여부 확인
            if (!request.headers.containsKey(HttpHeaders.AUTHORIZATION)) {
                return@GatewayFilter onError(exchange, "No Authorization header", HttpStatus.UNAUTHORIZED)
            }

            // 3. 'Bearer ' 접두사로 시작하는 토큰 추출
            val authorizationHeader = request.headers[HttpHeaders.AUTHORIZATION]!![0]
            if (!authorizationHeader.startsWith("Bearer ")) {
                return@GatewayFilter onError(exchange, "Invalid token format", HttpStatus.UNAUTHORIZED)
            }
            val token = authorizationHeader.substring(7)
            
            // 4. 토큰 유효성 검증
            try {
                // (만료 여부 등 상세 검증은 JwtUtil 내부에 구현 가능)
                val memberId = jwtUtil.getMemberId(token)
                
                // 5. 검증 성공 시, 요청에 'X-Member-Id' 헤더 추가
                val modifiedRequest = request.mutate()
                    .header("X-Member-Id", memberId.toString())
                    .build()
                
                chain.filter(exchange.mutate().request(modifiedRequest).build())
            } catch (e: JwtException) {
                onError(exchange, "Invalid token: ${e.message}", HttpStatus.UNAUTHORIZED)
            }
        }
    }
    
    private fun onError(exchange: ServerWebExchange, err: String, httpStatus: HttpStatus): Mono<Void> {
        val response = exchange.response
        response.statusCode = httpStatus
        // (로그 남기는 로직 추가)
        return response.setComplete()
    }
}
```

### 3\. `application.yml`에 커스텀 필터 적용

마지막으로, 이 `AuthenticationFilter`를 **모든 라우트에 기본으로 적용**하도록 `default-filters` 설정을 추가합니다.

```yaml
# gateway-service/src/main/resources/application.yml

spring:
  cloud:
    gateway:
      # --- 모든 라우트에 기본으로 적용될 필터 ---
      default-filters:
        # 커스텀 필터 이름은 'Filter' 접미사를 제외하고 카멜케이스로 작성
        - AuthenticationFilter
        
      routes:
        # ... (라우트 정의) ...
```

이제 게이트웨이를 재시작하면, 로그인/가입을 제외한 모든 요청은 `AuthenticationFilter`를 거치게 됩니다. 유효한 JWT 없이는 그 어떤 마이크로서비스에도 접근할 수 없게 되었습니다. 우리 MSA의 '인증' 문제가 중앙에서 깔끔하게 해결되었습니다.