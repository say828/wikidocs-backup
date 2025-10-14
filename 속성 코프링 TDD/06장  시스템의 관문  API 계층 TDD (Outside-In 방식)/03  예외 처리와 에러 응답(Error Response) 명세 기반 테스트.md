## 03\. 예외 처리와 에러 응답(Error Response) 명세 기반 테스트

훌륭한 API는 성공의 순간뿐만 아니라, 실패의 순간에도 클라이언트와 명확하게 소통한다. 사용자를 찾을 수 없을 때(`404 Not Found`), 요청 데이터가 잘못되었을 때(`400 Bad Request`), 서버 내부에서 예기치 못한 문제가 발생했을 때(`500 Internal Server Error`), API는 어떤 일이 일어났는지 클라이언트가 이해할 수 있는 일관된 형식의 \*\*에러 응답(Error Response)\*\*을 내려주어야 한다.

이 '에러 응답' 역시 API 명세의 중요한 일부다. TDD를 통해 우리는 특정 예외 상황이 발생했을 때, 약속된 상태 코드와 에러 메시지 형식이 정확하게 반환되는지를 **명세 기반으로 테스트**해야 한다. 이는 API의 신뢰성과 사용자 경험을 결정하는 매우 중요한 과정이다.

### **나쁜 예: 일관성 없는 예외 처리**

가장 흔한 안티패턴은 컨트롤러의 각 메소드에서 `try-catch` 블록을 사용하여 예외를 개별적으로 처리하는 것이다. 이는 다음과 같은 문제를 낳는다.

  * **코드 중복:** 유사한 예외 처리 코드가 여러 컨트롤러에 흩어져 유지보수를 어렵게 만든다.
  * **일관성 파괴:** 개발자마다 다른 방식으로 에러 응답을 만들게 되어, 어떤 에러는 `{ "error": "message" }` 형태로, 다른 에러는 `{ "errorMessage": "message" }` 형태로 반환되는 등 API의 일관성이 깨진다.

### **전문가의 해법: `@RestControllerAdvice`를 이용한 중앙 집중식 예외 처리**

스프링은 이러한 문제를 우아하게 해결할 수 있는 `@RestControllerAdvice`와 `@ExceptionHandler`라는 강력한 무기를 제공한다. `@RestControllerAdvice`는 애플리케이션 전역에서 발생하는 예외를 한 곳에서 가로채어 공통된 방식으로 처리할 수 있게 해주는 AOP(관점 지향 프로그래밍) 기반의 기능이다.

우리는 TDD를 통해 이 중앙 예외 처리기가 올바르게 동작하는지를 먼저 검증할 것이다.

**1. 실패하는 테스트로 에러 응답 명세를 작성하라 (RED)**

먼저, "존재하지 않는 사용자 ID로 조회하면, 404 상태 코드와 함께 표준 에러 형식의 응답이 와야 한다"는 테스트를 작성한다.

```kotlin
// ErrorResponse.kt (표준 에러 응답 DTO)
data class ErrorResponse(
    val errorCode: String,
    val message: String
)

// UserControllerTest.kt 내부
given("존재하지 않는 사용자 ID가 주어졌을 때") {
    val nonExistentUserId = 999L
    
    // Service 계층이 UserNotFoundException을 던지도록 Stubbing
    every { userService.getUserById(nonExistentUserId) } throws UserNotFoundException("사용자를 찾을 수 없습니다: id=$nonExistentUserId")

    `when`("GET /users/{id} API를 호출하면") {
        val actions = mockMvc.perform(get("/users/$nonExistentUserId"))

        then("404 Not Found 상태 코드와 표준 에러 응답을 반환해야 한다") {
            actions
                .andExpect(status().isNotFound)
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.errorCode").value("USER_NOT_FOUND"))
                .andExpect(jsonPath("$.message").value("사용자를 찾을 수 없습니다: id=$nonExistentUserId"))
                .andDo(print())
        }
    }
}
```

`@RestControllerAdvice`가 없으므로, 이 테스트는 `UserNotFoundException`을 처리하지 못하고 500 Internal Server Error를 반환하며 실패한다.

**2. `@RestControllerAdvice`로 테스트를 통과시켜라 (GREEN)**

이제 이 테스트를 통과시키는 전역 예외 처리기를 작성한다.

```kotlin
// GlobalExceptionHandler.kt
@RestControllerAdvice
class GlobalExceptionHandler {

    // UserNotFoundException 타입의 예외가 발생하면 이 메소드가 실행된다.
    @ExceptionHandler(UserNotFoundException::class)
    fun handleUserNotFoundException(e: UserNotFoundException): ResponseEntity<ErrorResponse> {
        val errorResponse = ErrorResponse(
            errorCode = "USER_NOT_FOUND",
            message = e.message ?: "사용자를 찾을 수 없습니다."
        )
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(errorResponse)
    }
    
    // 다른 종류의 예외 핸들러들을 여기에 추가...
    @ExceptionHandler(MethodArgumentNotValidException::class) // @Valid 실패 시
    fun handleValidationExceptions(e: MethodArgumentNotValidException): ResponseEntity<ErrorResponse> {
        // ...
        return ResponseEntity.badRequest().body(...)
    }
}
```

이제 다시 테스트를 실행하면, `UserNotFoundException`이 발생했을 때 `GlobalExceptionHandler`가 이를 가로채 우리가 명세한 정확한 JSON 응답과 404 상태 코드를 반환하므로 테스트가 성공한다.

-----

TDD를 통한 예외 처리 테스트는 API의 '불행한 경로(unhappy path)'에 대한 가장 확실한 안전망을 제공한다. `@RestControllerAdvice`를 활용한 중앙 집중식 예외 처리 전략은 우리 API가 어떤 예외 상황에서도 항상 예측 가능하고 일관된 방식으로 응답하도록 보장한다. 이는 API의 안정성과 클라이언트 개발자의 개발 경험을 모두 향상시키는, 프로페셔널한 API 설계의 필수 요소다.

이것으로 시스템의 관문인 API 계층의 TDD 여정을 마친다. 우리는 이제 사용자의 요청을 받아들이고, 유효성을 검사하며, 성공과 실패의 모든 경우에 대해 명세에 따라 정확히 응답하는 견고한 API를 구축할 수 있게 되었다. 다음 장에서는 현대 아키텍처의 가장 큰 난제 중 하나인 비동기 세계를 TDD로 정복하는 방법을 탐험할 것이다.