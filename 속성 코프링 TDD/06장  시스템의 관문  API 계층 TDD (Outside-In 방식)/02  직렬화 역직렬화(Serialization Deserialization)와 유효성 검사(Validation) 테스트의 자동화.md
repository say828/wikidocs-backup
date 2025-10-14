## 02\. 직렬화/역직렬화(Serialization/Deserialization)와 유효성 검사(Validation) 테스트의 자동화

API 컨트롤러의 가장 중요한 책임 중 하나는 시스템의 '문지기' 역할이다. 문지기는 두 가지 일을 해야 한다. 첫째, 외부에서 온 손님(HTTP 요청)의 언어(JSON)를 내부에서 사용하는 언어(DTO 객체)로 정확히 통역해야 한다(**역직렬화**). 둘째, 수상한 손님(유효하지 않은 데이터)은 성 안으로 들여보내지 않고 문전박대해야 한다(**유효성 검사**).

TDD를 통해 우리는 이 두 가지 핵심 책임이 항상 올바르게 동작함을 보장해야 한다. `MockMvc`는 이 문지기의 행동을 검증하는 데 최적화된 도구를 제공한다.

### **JSON과의 대화: 직렬화/역직렬화 테스트**

스프링 부트는 Jackson 라이브러리를 통해 대부분의 JSON 변환을 자동으로 처리해주지만, 이 '마법'을 맹신해서는 안 된다. 다음과 같은 미묘한 문제들이 언제든 발생할 수 있다.

  * JSON의 `snake_case` 필드(`user_name`)와 코틀린 DTO의 `camelCase` 프로퍼티(`userName`)가 제대로 매핑되지 않는 문제
  * 날짜/시간(`LocalDateTime`) 포맷이 기대와 다르게 변환되는 문제
  * 직접 만든 Custom Serializer/Deserializer가 의도대로 동작하지 않는 문제

이러한 문제를 방지하기 위해, 우리는 API 테스트에서 직렬화/역직렬화 과정을 명시적으로 검증해야 한다.

  * **역직렬화 (Deserialization: JSON → DTO) 테스트:** 우리가 앞 절에서 작성했던 `POST /users` 테스트는 그 자체로 훌륭한 역직렬화 테스트다. `mockMvc.perform`에 `.content(requestJson)`로 보낸 JSON 문자열이 컨트롤러의 `@RequestBody` 파라미터인 `CreateUserRequest` DTO로 올바르게 변환되었음을 테스트의 최종 성공을 통해 증명하기 때문이다.

  * **직렬화 (Serialization: DTO → JSON) 테스트:** 반대로, 우리 시스템이 응답으로 보내는 DTO가 클라이언트가 기대하는 정확한 JSON 형태로 변환되는지 검증해야 한다. `GET /users/{id}`와 같은 조회 API가 좋은 예시다.

**예시: 사용자 조회 API의 응답 직렬화 테스트**

```kotlin
@Test
fun `사용자 ID로 조회하면, 해당 사용자의 정보가 정확한 JSON 형식으로 반환된다`() {
    // given: Service 계층은 특정 UserResponse DTO를 반환하도록 Mocking/Stubbing
    val userId = 1L
    val creationTime = LocalDateTime.now()
    val expectedResponse = UserResponse(id = userId, name = "TDD Master", email = "test@tdd.com", createdAt = creationTime)
    every { userService.getUserById(userId) } returns expectedResponse

    // when & then
    mockMvc.perform(get("/users/$userId"))
        .andExpect(status().isOk)
        .andExpect(content().contentType(MediaType.APPLICATION_JSON))
        .andExpect(jsonPath("$.id").value(userId))
        .andExpect(jsonPath("$.name").value("TDD Master"))
        // 날짜/시간 포맷이 ISO-8601 표준에 맞게 직렬화되었는지 검증
        .andExpect(jsonPath("$.createdAt").value(creationTime.toString()))
}
```

### **잘못된 데이터를 막아라: 유효성 검사 테스트의 자동화**

`@Valid` 어노테이션과 `@NotBlank`, `@Email`, `@Min` 등 `jakarta.validation` 어노테이션을 사용한 유효성 검사는 시스템을 보호하는 첫 번째 방어선이다. 우리는 이 방어선에 빈틈이 없는지 체계적으로 테스트해야 한다. 온갖 종류의 잘못된 데이터를 테스트 케이스로 만드는 작업은 Kotest의 데이터 주도 테스트(`withData`)를 사용하면 매우 간결하고 효율적으로 자동화할 수 있다.

**예시: 사용자 생성 요청에 대한 유효성 검사 DDT**

먼저, `CreateUserRequest` DTO에 유효성 검사 어노테이션을 추가한다. (`@field:`는 코틀린 프로퍼티의 자바 필드에 어노테이션을 적용하기 위해 필요하다.)

```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "이름은 비어 있을 수 없습니다.")
    val name: String,

    @field:Email(message = "올바른 이메일 형식이어야 합니다.")
    val email: String
)
```

이제 다양한 실패 시나리오를 테스트 코드로 명세한다.

```kotlin
// UserControllerTest.kt 내부
context("사용자 생성 요청 데이터 유효성 검사") {
    withData(
        // name, requestJson, expectedErrorMessage
        "이름이 비어 있으면 실패한다" to Pair("""{"name":"", "email":"test@tdd.com"}""", "이름은 비어 있을 수 없습니다."),
        "이름이 공백이면 실패한다" to Pair("""{"name":" ", "email":"test@tdd.com"}""", "이름은 비어 있을 수 없습니다."),
        "이메일 형식이 틀리면 실패한다" to Pair("""{"name":"TDD", "email":"invalid-email"}""", "올바른 이메일 형식이어야 합니다.")
    ) { (requestJson, expectedErrorMessage) ->
        mockMvc.perform(post("/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content(requestJson))
            .andExpect(status().isBadRequest) // 400 Bad Request를 기대
            // Spring Boot의 기본 에러 응답 본문에 에러 메시지가 포함되었는지 검증
            .andExpect(jsonPath("$.errors[0].defaultMessage").value(expectedErrorMessage))
    }
}
```

데이터 주도 테스트를 통해 우리는 단 하나의 테스트 로직으로 여러 유효성 검사 규칙을 체계적으로 검증했다. 새로운 규칙이 추가되면 테스트 데이터 목록에 한 줄만 더 추가하면 된다.

API 계층에서의 직렬화 및 유효성 검사 테스트는 우리 시스템의 '입구'와 '출구'를 단단히 지키는 파수꾼을 세우는 것과 같다. 이 테스트들을 통해 우리는 오직 깨끗하고 올바른 데이터만이 우리 시스템 내부로 들어오고, 약속된 형식의 데이터만이 외부로 나간다는 것을 보장할 수 있다.