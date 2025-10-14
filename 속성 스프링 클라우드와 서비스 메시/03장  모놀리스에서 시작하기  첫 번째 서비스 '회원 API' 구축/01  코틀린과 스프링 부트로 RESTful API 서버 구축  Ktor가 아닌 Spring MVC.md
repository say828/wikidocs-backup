## 코틀린과 스프링 부트로 RESTful API 서버 구축: Ktor가 아닌 Spring MVC

02장에서 설계한 '청사진'을 바탕으로, 이제 첫 번째 코드('회원' 모놀리스)를 작성할 시간입니다.

우리의 주력 언어는 코틀린입니다. 그렇다면 코틀린 애플리케이션을 만들 때, 코틀린 네이티브(Native) 프레임워크인 `Ktor`가 아닌 스프링 부트(Spring Boot)의 `Spring MVC`를 선택한 이유는 무엇일까요?

  * **Ktor:** 코루틴 기반의 경량화되고 유연한 비동기 프레임워크입니다. 매우 훌륭한 선택지이지만, 엔터프라이즈급 생태계가 상대적으로 부족합니다.
  * **Spring MVC:** 자바 진영에서 20년간 검증된, '엔터프라이즈 아키텍처'의 사실상 표준(de facto standard)입니다.

우리가 **Spring MVC**를 선택한 이유는 명확합니다.

1.  **검증된 통합 생태계 (Proven Ecosystem):** 이 책의 핵심 주제인 `Spring Data JPA`(DB), `Spring Security`(인증), `Spring Cloud`(MSA)와의 완벽한 통합을 제공하는 것은 스프링 부트가 유일합니다. Ktor로 이 모든 것을 조합하는 것은 몇 배의 노력이 듭니다.
2.  **"모놀리스 퍼스트" 전략과의 부합:** Spring MVC의 '1 요청-1 스레드' 기반 블로킹(Blocking) 모델은, 모놀리스 환경에서 CRUD API를 개발하는 **가장 빠르고, 가장 단순하며, 가장 직관적인** 방법입니다.
3.  **명확한 학습 경로:** 우리는 01장에서 '블로킹' 모델의 한계를 이미 학습했습니다. 03장에서 이 '블로킹' MVC 모델로 빠르게 서비스를 구축하고, 12장에서 `Spring WebFlux`와 `코루틴` 기반의 '논블로킹' 모델로 마이그레이션할 것입니다. 이 '고통'과 '진화'의 과정을 경험하는 것이 이 책의 핵심 학습 목표입니다.

-----

### 프로젝트 설정 (`build.gradle.kts`)

먼저, 코틀린 + 스프링 부트 프로젝트를 위한 `build.gradle.kts`의 핵심 의존성을 설정합니다.

```kotlin
// build.gradle.kts
plugins {
    id("org.springframework.boot") version "3.2.5" // (주: 현시점 최신 안정화 버전 기준)
    id("io.spring.dependency-management") version "1.1.5"
    kotlin("jvm") version "1.9.24"
    kotlin("plugin.spring") version "1.9.24" // 1. 스프링 클래스/함수를 'open'으로 만들기 위한 플러그인
    kotlin("plugin.jpa") version "1.9.24"    // 2. JPA @Entity를 위한 'all-open', 'no-arg' 플러그인
}

dependencies {
    // 3. Spring MVC, 내장 Tomcat, JSON(Jackson)을 포함하는 핵심 스타터
    implementation("org.springframework.boot:spring-boot-starter-web")
    
    // 4. JPA (다음 03.02절에서 즉시 사용)
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    
    // 5. 코틀린 data class를 Jackson이 올바르게 (역)직렬화하기 위한 필수 모듈
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    
    // 코틀린 표준 라이브러리 및 리플렉션
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlin:kotlin-stdlib")

    // (테스트 및 DB 의존성은 03.03절에서 추가)
}
```

  * `kotlin("plugin.spring")`와 `kotlin("plugin.jpa")`는 코틀린의 `final` 클래스 특성을 JPA와 스프링 AOP(트랜잭션 등)가 프록시(Proxy)할 수 있도록 `open`으로 변경해 주는 필수 플러그인입니다. (01장 02절의 함정 참조)

-----

### 메인 애플리케이션 실행

스프링 부트의 진입점을 만듭니다.

```kotlin
// /src/main/kotlin/com/ecommerce/member/MemberApplication.kt
package com.ecommerce.member

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class MemberApplication

fun main(args: Array<String>) {
    runApplication<MemberApplication>(*args)
}
```

### '회원 API' 계층(Layer) 설계

우리는 표준적인 '계층형 아키텍처(Layered Architecture)'를 따릅니다.

1.  **Controller (Web Layer):** HTTP 요청을 받고, DTO로 변환하며, `Service`를 호출합니다.
2.  **Service (Business Layer):** 비즈니스 로직을 수행합니다.
3.  **Repository (Data Layer):** DB와 통신합니다. (03.02절에서 구현)

#### 1\. DTO (Data Transfer Objects)

01장에서 배운 `data class`를 활용하여 요청/응답 객체를 정의합니다.

```kotlin
// /src/main/kotlin/com/ecommerce/member/api/MemberDto.kt
package com.ecommerce.member.api

// 회원 가입 요청 DTO
data class JoinMemberRequest(
    val email: String,
    val name: String,
    val address: String,
    // (패스워드 등은 04장 인증에서 추가)
)

// 회원 정보 응답 DTO
data class MemberResponse(
    val id: Long,
    val email: String,
    val name: String,
)
```

#### 2\. Controller & Service (API & Business Skeletons)

`@RestController`를 사용하여 '회원' API의 엔드포인트를 정의하고, `@Service`로 비즈니스 로직을 위임합니다.

```kotlin
// /src/main/kotlin/com/ecommerce/member/api/MemberController.kt
package com.ecommerce.member.api

import com.ecommerce.member.service.MemberService
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/v1/members")
class MemberController(
    private val memberService: MemberService // 1. 생성자 주입
) {

    @PostMapping("/join")
    fun join(@RequestBody request: JoinMemberRequest): MemberResponse {
        // 2. 요청 DTO를 Service로 전달
        return memberService.join(request)
    }

    @GetMapping("/{memberId}")
    fun getMember(@PathVariable memberId: Long): MemberResponse {
        // 3. Path Variable을 Service로 전달
        return memberService.getMember(memberId)
    }
}
```

```kotlin
// /src/main/kotlin/com/ecommerce/member/service/MemberService.kt
package com.ecommerce.member.service

import com.ecommerce.member.api.JoinMemberRequest
import com.ecommerce.member.api.MemberResponse
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional(readOnly = true) // 1. 기본적으로 읽기 전용 트랜잭션
class MemberService {
    
    // (JPA Repository는 03.02에서 주입)

    @Transactional // 2. 쓰기 트랜잭션 추가
    fun join(request: JoinMemberRequest): MemberResponse {
        // (03.02에서 실제 구현)
        // TODO: email 중복 검사
        // TODO: request DTO -> Member Entity 변환
        // TODO: memberRepository.save(member)
        // TODO: member Entity -> MemberResponse DTO 변환 후 반환
        
        println("회원 가입 로직 처리: ${request.name}")
        // 3. 임시 반환 데이터
        return MemberResponse(id = 1L, email = request.email, name = request.name)
    }

    fun getMember(memberId: Long): MemberResponse {
        // (03.02에서 실제 구현)
        // TODO: memberRepository.findByIdOrNull(memberId)
        // TODO: Entity -> DTO 변환 후 반환 (Entity가 없으면 예외 처리)
        
        println("회원 조회 로직 처리: $memberId")
        // 3. 임시 반환 데이터
        return MemberResponse(id = memberId, email = "test@email.com", name = "Test User")
    }
}
```

-----

이제 `http://localhost:8080`으로 스프링 부트 애플리케이션을 실행하면, 우리는 '회원' API의 '껍데기(Skeleton)'가 동작하는 것을 확인할 수 있습니다.

다음 절에서는 `MemberService`의 `// TODO` 주석을 실제 코드로 채울 차례입니다. `Spring Data JPA`와 코틀린 `@Entity`를 사용하여 실제 데이터베이스에 '회원' 정보를 저장하고 조회해 보겠습니다.