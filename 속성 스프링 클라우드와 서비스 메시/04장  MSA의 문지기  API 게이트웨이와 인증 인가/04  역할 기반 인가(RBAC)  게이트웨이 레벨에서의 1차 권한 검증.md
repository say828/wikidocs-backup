## 역할 기반 인가(RBAC): 게이트웨이 레벨에서의 1차 권한 검증

04장 03절에서 우리는 JWT를 통해 '인증(Authentication)' 문제를 해결했습니다. '인증'은 \*\*"당신은 누구인가?(Who are you?)"\*\*에 대한 답입니다. 게이트웨이는 이제 `X-Member-Id` 헤더를 통해 요청자가 우리 시스템의 유효한 '회원'임을 보장합니다.

하지만 이것만으로는 부족합니다. 다음 질문에 답해야 합니다.

> **"그래서 이 회원이, 이 작업을 수행할 '권한'이 있는가?"**

이것이 바로 \*\*'인가(Authorization)'\*\*의 영역입니다. '인가'는 \*\*"당신은 무엇을 할 수 있는가?(What are you allowed to do?)"\*\*에 대한 답입니다.

예를 들어, 일반 `USER`는 자신의 주문 내역(`GET /api/v1/orders/my-orders`)은 조회할 수 있지만, 관리자(ADMIN) 전용인 '전체 회원 목록 조회'(`GET /api/v1/admin/members`) API에 접근해서는 안 됩니다.

-----

### 게이트웨이의 역할: 1차 검문소

물론, "123번 회원이 456번 주문을 취소할 수 있는가?"와 같은 **세분화된(Fine-grained) 권한**은 최종적으로 `order-service`가 해당 주문의 소유권을 확인하여 처리해야 합니다.

하지만 `/api/v1/admin/**` 경로와 같이, 특정 \*\*'역할(Role)'\*\*을 가진 사용자 그룹 전체에 적용되는 **포괄적인(Coarse-grained) 권한**은 모든 서비스가 중복으로 검사할 필요가 없습니다. 이 또한 게이트웨이에서 처리하기 완벽한 '횡단 관심사'입니다.

게이트웨이는 **'1차 검문소'** 역할을 수행하여, 애초에 권한이 없는 요청이 내부 시스템으로 들어오는 것 자체를 차단합니다. 이는 내부 서비스들의 부담을 줄이고, 보안을 한층 강화합니다.

-----

### 구현 단계

#### 1\. JWT에 '역할(Role)' 정보 추가 (`member-service`)

'인가'를 위해서는 먼저 사용자의 '역할'을 알아야 합니다. 이 정보는 인증 주체인 `member-service`가 JWT를 발급할 때 \*\*'roles'라는 커스텀 클레임(Custom Claim)\*\*에 담아주어야 합니다.

```kotlin
// (참고) member-service의 JWT 발급 로직 (가상 코드)
fun createToken(member: Member): String {
    val claims = Jwts.claims()
        .subject(member.id.toString())
        .add("roles", member.roles) // ["USER"], 또는 ["USER", "ADMIN"]
        .build()
    
    return Jwts.builder()
        .claims(claims)
        // ... setIssuedAt, setExpiration, signWith ...
        .compact()
}
```

#### 2\. 게이트웨이에서 '역할' 정보 파싱 (`gateway-service`)

`gateway-service`의 `JwtUtil`에 JWT에서 `roles` 클레임을 추출하는 기능을 추가합니다.

```kotlin
// gateway-service/src/main/kotlin/com/ecommerce/gateway/util/JwtUtil.kt
// ... (기존 코드) ...
class JwtUtil(...) {
    
    // ... (getAllClaims, getMemberId) ...

    /**
     * 토큰에서 역할(roles) 정보 추출
     */
    @Suppress("UNCHECKED_CAST")
    fun getRoles(token: String): List<String> {
        return getAllClaims(token)["roles"] as List<String>? ?: emptyList()
    }
}
```

#### 3\. '인가' 필터 로직 추가 (`gateway-service`)

이제 04장 03절에서 만든 `AuthenticationFilter`에 '인가' 로직을 추가합니다. 단순히 코드를 하드코딩하는 대신, `application.yml`에 경로별 권한 설정을 정의하여 유연성을 높이겠습니다.

먼저, `application.yml`에 경로별 필요 권한을 정의합니다.

```yaml
# gateway-service/src/main/resources/application.yml
# ... (기존 설정) ...

# 1. 경로별 권한 설정
security:
  # 'role' 키를 가진 경로 목록
  required-roles:
    # '/api/v1/admin/**' 경로는 ADMIN 역할이 필요
    - path: /api/v1/admin/**
      role: ADMIN
    # '/api/v1/orders/**' 경로는 USER 역할이 필요
    - path: /api/v1/orders/**
      role: USER
    # (여기에 명시되지 않은 경로는 인증만 되면 통과)
```

이 설정을 읽어올 `ConfigurationProperties` 클래스를 만듭니다.

```kotlin
// gateway-service/src/main/kotlin/com/ecommerce/gateway/config/SecurityProperties.kt
package com.ecommerce.gateway.config

import org.springframework.boot.context.properties.ConfigurationProperties

@ConfigurationProperties("security")
data class SecurityProperties(
    val requiredRoles: List<RouteRoleConfig> = emptyList()
)

data class RouteRoleConfig(
    val path: String,
    val role: String,
)
```

`AuthenticationFilter`를 수정하여 이 설정을 기반으로 '인가' 로직을 수행합니다.

```kotlin
// gateway-service/src/main/kotlin/com/ecommerce/gateway/filter/AuthenticationFilter.kt
// ... (import문) ...
import com.ecommerce.gateway.config.SecurityProperties // 1. 설정 클래스 import
import org.springframework.util.AntPathMatcher // 2. 경로 패턴 매칭을 위한 유틸리티

@Component
class AuthenticationFilter(
    private val jwtUtil: JwtUtil,
    private val securityProperties: SecurityProperties, // 3. 설정 주입
) : AbstractGatewayFilterFactory<AuthenticationFilter.Config>(Config::class.java) {
    
    private val pathMatcher = AntPathMatcher() // 4. 경로 패턴 매처

    // ... (Config 클래스, onError 함수) ...

    override fun apply(config: Config): GatewayFilter {
        return GatewayFilter { exchange, chain ->
            val request = exchange.request
            // ... (기존: 공개 엔드포인트 체크, 헤더 체크, 토큰 추출) ...
            
            val token = authorizationHeader.substring(7)
            
            try {
                // --- 1. 인증 (Authentication) ---
                val memberId = jwtUtil.getMemberId(token)
                
                // --- 2. 인가 (Authorization) ---
                val memberRoles = jwtUtil.getRoles(token)
                val requiredRole = getRequiredRoleForPath(request.uri.path)
                
                if (requiredRole != null && !memberRoles.contains(requiredRole)) {
                    return@GatewayFilter onError(exchange, "Required role not satisfied", HttpStatus.FORBIDDEN) // 403 Forbidden
                }
                
                // --- 3. 성공 처리 ---
                val modifiedRequest = request.mutate()
                    .header("X-Member-Id", memberId.toString())
                    .build()
                
                chain.filter(exchange.mutate().request(modifiedRequest).build())
            } catch (e: JwtException) {
                onError(exchange, "Invalid token: ${e.message}", HttpStatus.UNAUTHORIZED)
            }
        }
    }
    
    /**
     * application.yml 설정을 기반으로 현재 경로에 필요한 역할을 찾아 반환
     */
    private fun getRequiredRoleForPath(path: String): String? {
        return securityProperties.requiredRoles
            .find { config -> pathMatcher.match(config.path, path) }
            ?.role
    }
}
```

마지막으로, `@ConfigurationProperties`를 사용하려면 메인 애플리케이션에 `@ConfigurationPropertiesScan` 어노테이션을 추가해야 합니다.

```
@ConfigurationPropertiesScan
@SpringBootApplication
class GatewayApplication

fun main(args: Array<String>) {
    runApplication<GatewayApplication>(*args)
}
```

-----

이제 우리 게이트웨이는 단순한 '인증'을 넘어, `ADMIN` 역할이 없는 사용자가 `/api/v1/admin` 경로로 접근하면 `403 Forbidden` 응답을 내리는 **'1차 인가'** 기능까지 갖추게 되었습니다.

이로써 내부 마이크로서비스들은 반복적인 역할 검사 로직에서 해방되어, 더욱 중요한 '비즈니스 로직'과 '세분화된 권한 검증'에 집중할 수 있게 되었습니다.