## 01\. Spring MockMvc 심층 분석: 요청(Request)부터 응답(Response)까지의 전체 흐름 검증

Outside-In TDD의 인수 테스트를 작성하기 위해, 우리는 실제 HTTP 서버를 띄우지 않고도 컨트롤러의 동작을 완벽하게 시뮬레이션할 수 있는 도구가 필요하다. 이것이 바로 **Spring MockMvc**가 존재하는 이유다. `MockMvc`는 이름에서 알 수 있듯이, 서블릿 컨테이너를 'Mocking'하여 실제 네트워크 연결 없이도 스프링 MVC 컨트롤러에 HTTP 요청을 보내고 응답을 검증할 수 있게 해주는 강력한 테스트 프레임워크다.

`MockMvc`를 사용하면 다음과 같은 시스템의 전체 흐름을 테스트할 수 있다.

`HTTP 요청 → DispatcherServlet → HandlerMapping → Controller → Service → ... → Controller → View/ResponseEntity → HTTP 응답`

즉, 우리는 HTTP 요청을 보내는 클라이언트의 입장에서부터 응답을 받는 순간까지, 스프링의 전체 요청 처리 파이프라인을 검증하게 된다.

### **MockMvc 테스트의 기본 구조**

`MockMvc` 테스트를 작성하기 위한 핵심 어노테이션은 `@SpringBootTest`와 `@AutoConfigureMockMvc`다.

  * `@SpringBootTest`: 실제 애플리케이션 실행과 거의 동일하게 모든 빈(Bean)을 포함한 전체 스프링 컨텍스트를 로드한다.
  * `@AutoConfigureMockMvc`: `MockMvc` 객체를 스프링 컨텍스트에 빈으로 등록하여, 테스트 클래스에서 `@Autowired`로 주입받을 수 있게 해준다.

<!-- end list -->

```kotlin
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.web.servlet.MockMvc

@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest(
    @Autowired private val mockMvc: MockMvc
) : BehaviorSpec({
    // ... 테스트 코드 ...
})
```

### **MockMvc API 3단계: `perform`, `andExpect`, `andDo`**

`MockMvc`를 사용한 테스트는 일반적으로 세 단계의 연쇄적인 메소드 호출로 구성된다.

1.  **`perform(requestBuilder)`: 요청 수행**

      * HTTP 요청을 생성하고 실행하는 단계. `MockMvcRequestBuilders` 클래스가 제공하는 `get`, `post`, `put`, `delete` 등의 정적 메소드를 사용하여 요청을 만든다.
      * 헤더 추가, 요청 본문(body) 설정 등 모든 HTTP 요청 정보를 여기서 구성한다.

2.  **`andExpect(resultMatcher)`: 응답 검증**

      * `perform`의 결과로 받은 `ResultActions` 객체에 대해 응답을 검증하는 단계. `MockMvcResultMatchers` 클래스가 제공하는 `status()`, `header()`, `content()`, `jsonPath()` 등의 정적 메소드를 사용하여 응답의 모든 측면을 검증할 수 있다.
      * 하나의 요청에 대해 여러 개의 `andExpect`를 연쇄적으로 호출하여 다양한 조건을 검증할 수 있다.

3.  **`andDo(resultHandler)`: 추가 작업 (Optional)**

      * 요청/응답의 전체 내용을 콘솔에 출력하여 디버깅하는 등 추가적인 작업을 수행하는 단계. `MockMvcResultHandlers.print()`가 가장 일반적으로 사용된다.

### **TDD 예시: 사용자 생성 API 인수 테스트**

이제 "이름과 이메일을 담아 `POST /users`로 요청하면 사용자가 생성된다"는 인수 테스트를 TDD로 작성해보자.

**1. 실패하는 인수 테스트 작성 (RED)**

```kotlin
// UserControllerTest.kt
import org.springframework.http.MediaType
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import org.springframework.test.web.servlet.result.MockMvcResultHandlers.print
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.*

// ... BehaviorSpec 내부 ...
given("새로운 사용자를 생성하기 위한 정보가 주어졌을 때") {
    val userName = "TDD Master"
    val userEmail = "test@tdd.com"
    val requestJson = """
        {
            "name": "$userName",
            "email": "$userEmail"
        }
    """.trimIndent()

    `when`("POST /users API를 호출하면") {
        val actions = mockMvc.perform(
            post("/users") // 1. perform: POST 요청 생성
                .contentType(MediaType.APPLICATION_JSON) // 요청의 Content-Type 설정
                .content(requestJson) // 요청 본문(body)에 JSON 데이터 추가
        )

        then("201 Created 상태 코드를 반환하고, 생성된 사용자 정보를 응답해야 한다") {
            actions
                .andExpect(status().isCreated) // 2. andExpect: HTTP 상태 코드 검증
                .andExpect(header().exists("Location")) // Location 헤더 존재 여부 검증
                .andExpect(jsonPath("$.id").exists()) // 응답 JSON 본문의 id 필드 존재 여부 검증
                .andExpect(jsonPath("$.name").value(userName)) // name 필드 값 검증
                .andExpect(jsonPath("$.email").value(userEmail)) // email 필드 값 검증
                .andDo(print()) // 3. andDo: 요청/응답 내용 출력
        }
    }
}
```

당연히 `/users` 엔드포인트가 없으므로 이 테스트는 404 Not Found를 반환하며 실패한다.

**2. 테스트를 통과시키는 최소한의 코드 작성 (GREEN)**

이제 이 인수 테스트를 통과시키기 위해 안쪽 루프 TDD(Controller, Service, Repository 단위 테스트)를 진행했다고 가정하고, 최종 구현 코드를 작성한다.

```kotlin
// UserController.kt
@RestController
class UserController(private val userService: UserService) {
    @PostMapping("/users")
    fun createUser(@RequestBody request: CreateUserRequest): ResponseEntity<UserResponse> {
        val userResponse = userService.createUser(request)
        val location = URI.create("/users/${userResponse.id}")
        return ResponseEntity.created(location).body(userResponse)
    }
}
// CreateUserRequest, UserResponse DTO 및 UserService 구현 생략...
```

이제 모든 계층의 코드가 준비되었으므로, 다시 인수 테스트를 실행하면 모든 `andExpect` 조건을 만족하며 성공적으로 통과한다.

`MockMvc`는 외부 세계와의 상호작용을 시작하는 인수 테스트를 작성하기 위한 완벽한 도구다. `jsonPath`와 같은 강력한 검증 기능을 통해, 우리는 API가 명세에 따라 정확하게 동작하는지를 매우 상세한 수준까지 보장할 수 있다. 이 견고한 인수 테스트의 안전망 위에서, 우리는 자신감을 가지고 내부 구현을 개선해 나갈 수 있다.