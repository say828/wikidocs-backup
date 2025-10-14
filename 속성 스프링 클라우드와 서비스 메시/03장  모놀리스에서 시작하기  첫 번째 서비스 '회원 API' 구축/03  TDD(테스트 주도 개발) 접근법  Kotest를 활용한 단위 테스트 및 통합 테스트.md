## TDD(테스트 주도 개발) 접근법: Kotest를 활용한 단위 테스트 및 통합 테스트

03장 02절에서 우리는 '회원 API'의 모든 기능을 구현했습니다. 하지만 "이 코드가 정말 의도한 대로 동작하는가?"를 어떻게 보장할 수 있을까요? 또한, 6개월 뒤 '회원 등급' 기능을 추가할 때, 이 기존 '회원 가입' 코드가 망가지지 않는다는 것은 어떻게 보장할까요?

그 해답이 바로 **테스트 코드**입니다.

이 책에서는 한 걸음 더 나아가, 코드를 작성하기 전에 테스트를 먼저 설계하는 **TDD(테스트 주도 개발, Test-Driven Development)** 접근법을 지향합니다. TDD는 단순히 버그를 잡는 것을 넘어, '동작하는 깔끔한 코드'를 작성하도록 유도하는 강력한 설계 도구입니다.

-----

### 왜 JUnit이 아닌 Kotest인가?

자바 진영의 표준 테스트 프레임워크는 `JUnit`입니다. 하지만 우리는 코틀린의 이점을 극대화하기 위해, 코틀린 네이티브 테스트 프레임워크인 \*\*`Kotest`\*\*를 사용합니다.

  * **가독성:** `assertEquals(expected, actual)`보다 `actual shouldBe expected`처럼 자연어에 가까운 문법을 제공합니다.
  * **유연한 구조:** `describe("상황") { context("조건") { it("결과") { ... } } }` (BehaviorSpec)처럼, 테스트의 '행위'와 '맥락'을 명확하게 구조화할 수 있습니다.
  * **강력한 Matcher:** `shouldThrow`, `shouldNotBeNull`, `list shouldContain "item"` 등 풍부한 검증 도구를 제공합니다.

-----

### 1\. 테스트 의존성 추가

먼저 `build.gradle.kts`에 `Kotest`와 `MockK` (코틀린을 위한 Mocking 라이브러리) 의존성을 추가합니다. `spring-boot-starter-test`는 통합 테스트를 위해 필요합니다.

```kotlin
// build.gradle.kts

dependencies {
    // ... (기존 implementation 의존성) ...

    // 1. 스프링 부트 기본 테스트 스타터 (JUnit, Mockito, MockMvc 등 포함)
    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude(group = "org.junit.vintage", module = "junit-vintage-engine")
    }

    // 2. Kotest Runner (JUnit5 위에서 동작)
    testImplementation("io.kotest:kotest-runner-junit5-jvm:5.8.1")
    // 3. Kotest Assertions (shouldBe 등)
    testImplementation("io.kotest:kotest-assertions-core-jvm:5.8.1")
    // 4. Spring과 Kotest를 연동하기 위한 라이브러리
    testImplementation("io.kotest.extensions:kotest-extensions-spring:1.1.3")

    // 5. 코틀린 네이티브 Mocking 라이브러리 (Mockito 대신 MockK 사용)
    testImplementation("io.mockk:mockk:1.13.10")
}

// Kotest가 JUnit5 플랫폼에서 실행되도록 설정
tasks.withType<Test> {
    useJUnitPlatform()
}
```

-----

### 2\. 단위 테스트 (Unit Test): `MemberService`

'단위 테스트'는 **다른 모든 외부 의존성을 '격리'(차단)하고,** 오직 '해당 클래스'의 로직만 테스트합니다. `MemberService`를 테스트할 때, `MemberRepository` (DB)는 '가짜(Mock)'로 대체해야 합니다.

여기서는 코틀린 네이티브 Mock 라이브러리인 `MockK`를 사용합니다.

```kotlin
// /src/test/kotlin/com/ecommerce/member/service/MemberServiceTest.kt
package com.ecommerce.member.service

import com.ecommerce.member.api.JoinMemberRequest
import com.ecommerce.member.api.toEntity
import com.ecommerce.member.domain.Member
import com.ecommerce.member.domain.MemberRepository
import io.kotest.assertions.throwables.shouldThrow
import io.kotest.core.spec.style.BehaviorSpec // 1. BehaviorSpec 사용
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import io.mockk.clearMocks
import io.mockk.every // 2. 'every'로 Mock의 행위 정의
import io.mockk.mockk // 3. 'mockk'로 Mock 객체 생성
import io.mockk.verify

// 테스트 대상 클래스(MemberService)에 final이 붙지 않도록
// build.gradle.kts의 all-open 플러그인 설정이 중요합니다.
class MemberServiceTest : BehaviorSpec({
    // 1. 테스트에 필요한 의존성(Repository)을 Mocking
    val memberRepository = mockk<MemberRepository>()

    // 2. 테스트 대상(SUT, System Under Test)인 MemberService에 Mock 주입
    val memberService = MemberService(memberRepository)

    // 3. (Best Practice) 각 테스트 케이스 실행 후 Mock 초기화
    afterTest {
        clearMocks(memberRepository)
    }
    
    // --- 테스트 케이스 1: 회원 가입 (성공) ---
    Given("회원 가입 요청 정보가 주어졌을 때") {
        val request = JoinMemberRequest(
            email = "test@email.com",
            name = "Test User",
            address = "Seoul"
        )
        val requestEntity = request.toEntity()
        
        // Mocking 정의:
        // 1. 이메일 중복 검사는 'null' (중복 없음)을 반환
        every { memberRepository.findByEmail(request.email) } returns null
        // 2. save(member)가 호출되면, id가 1L로 채워진 member를 반환
        every { memberRepository.save(any()) } returns Member(
            email = request.email,
            name = request.name,
            address = request.address
        ).apply { id = 1L } // (실제로는 id가 private set이 아니지만, 테스트 편의를 위해 임시 설정)
        // (더 좋은 방법은 Member의 private id를 reflection으로 세팅하는 것입니다.)

        When("회원 가입을 시도하면") {
            val response = memberService.join(request)

            Then("회원 정보가 반환되고, ID는 null이 아니어야 한다") {
                response.id shouldBe 1L
                response.email shouldBe request.email
                
                // save 함수가 1번 호출되었는지 검증
                verify(exactly = 1) { memberRepository.save(any()) }
            }
        }
    }

    // --- 테스트 케이스 2: 회원 가입 (실패 - 이메일 중복) ---
    Given("이미 존재하는 이메일로 회원 가입 요청이 주어졌을 때") {
        val request = JoinMemberRequest("duplicate@email.com", "Test", "Seoul")
        
        // Mocking 정의: 이메일 중복 검사 시 'Member' 객체 반환
        every { memberRepository.findByEmail(request.email) } returns Member("dup", "dup", "dup")

        When("회원 가입을 시도하면") {
            Then("IllegalArgumentException 예외가 발생해야 한다") {
                val exception = shouldThrow<IllegalArgumentException> {
                    memberService.join(request)
                }
                exception.message shouldBe "이미 가입된 이메일입니다: ${request.email}"
                
                // save 함수는 절대 호출되면 안 됨
                verify(exactly = 0) { memberRepository.save(any()) }
            }
        }
    }
    
    // (getMember에 대한 테스트 케이스도 유사하게 작성...)
})
```

-----

### 3\. 통합 테스트 (Integration Test): `MemberController`

'통합 테스트'는 **'API 요청부터 DB 저장까지'** 시스템의 전체 흐름을 테스트합니다. 여기서는 `MockMvc`를 사용하여 실제 HTTP 요청을 시뮬레이션하고, DB는 `H2` 같은 **인메모리 DB**를 사용합니다.

```kotlin
// /src/test/kotlin/com/ecommerce/member/api/MemberControllerTest.kt
package com.ecommerce.member.api

import com.ecommerce.member.domain.MemberRepository
import com.fasterxml.jackson.databind.ObjectMapper
import io.kotest.core.spec.style.DescribeSpec
import io.kotest.extensions.spring.SpringExtension
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.http.MediaType
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.post
import org.springframework.transaction.annotation.Transactional

@SpringBootTest // 1. 전체 스프링 컨텍스트를 로드 (통합 테스트)
@AutoConfigureMockMvc // 2. MockMvc 주입
@Transactional // 3. 테스트 후 롤백을 위한 트랜잭션 (DB 상태 유지)
@ActiveProfiles("test") // 4. 'application-test.yml' (H2 DB 설정) 사용
class MemberControllerTest(
    private val mockMvc: MockMvc,
    private val objectMapper: ObjectMapper, // JSON 직렬화/역직렬화
    private val memberRepository: MemberRepository // 5. DB 상태 검증용
) : DescribeSpec({
    
    // Kotest와 SpringBootTest를 연동
    override fun extensions() = listOf(SpringExtension)

    // 각 테스트 케이스 실행 전 DB 초기화 (선택적)
    beforeEach {
        memberRepository.deleteAll()
    }

    describe("POST /api/v1/members/join") {
        context("유효한 회원 가입 요청이 주어지면") {
            val request = JoinMemberRequest(
                email = "integration@email.com",
                name = "Integration User",
                address = "Pangyo"
            )
            val jsonBody = objectMapper.writeValueAsString(request)

            it("200 OK와 함께 회원 정보를 반환하고, DB에 저장된다") {
                // When: 실제 HTTP POST 요청 시뮬레이션
                mockMvc.post("/api/v1/members/join") {
                    contentType = MediaType.APPLICATION_JSON
                    content = jsonBody
                }.andExpect {
                    // Then 1: HTTP 응답 검증
                    status { isOk() } // 200 OK
                    jsonPath("$.id") { value(1L) } // (DB ID가 1로 시작)
                    jsonPath("$.email") { value(request.email) }
                }

                // Then 2: DB 상태 검증
                val savedMember = memberRepository.findByEmail(request.email)
                savedMember shouldNotBe null
                savedMember?.name shouldBe request.name
            }
        }
        
        // (중복 이메일 요청 시 400 Bad Request 반환 테스트 등...)
    }
})
```

  * 위 테스트를 위해 `src/test/resources/application-test.yml` 파일에 H2 인메모리 DB 설정을 추가해야 합니다.