## 02\. MockK 완벽 가이드: 단순 Mock을 넘어선 Spy, Capturing, Hierarchical Stubbing

Kotest가 테스트의 구조와 표현력을 담당한다면, **MockK**는 테스트 대상의 협력 객체들을 정교하게 제어하는 역할을 맡는다. 1장에서 우리는 DIP를 통해 의존성을 분리해야 테스트 용이성이 높아진다고 배웠다. MockK는 바로 이 분리된 의존성을 테스트 환경에서 자유자재로 다룰 수 있게 해주는 강력한 도구다.

코틀린은 `final` 클래스가 기본이며, `object`, 최상위 함수, 확장 함수 등 기존 자바 Mocking 프레임워크인 Mockito가 다루기 까다로운 언어적 특성을 많이 가지고 있다. MockK는 처음부터 코틀린을 위해 설계되어 이러한 문제들을 완벽하게 해결하며, 코틀린 DSL을 활용한 직관적인 문법을 제공한다.

### **기본 중의 기본: `mockk`, `every`, `returns`, `verify`**

MockK의 핵심은 네 가지 함수로 요약할 수 있다.

1.  **`mockk<T>()`**: 특정 타입 `T`에 대한 목(Mock) 객체를 생성한다.
2.  **`every { ... }`**: 특정 목 객체의 메소드 호출을 가로채서, 어떤 행동을 정의할지 지정하는 DSL 블록을 시작한다.
3.  **`returns ...`**: `every` 블록 안에서, 가로챈 메소드가 반환할 값을 지정한다.
4.  **`verify { ... }`**: 테스트가 실행된 후, 특정 목 객체의 메소드가 예상대로 호출되었는지(행위 기반 검증) 확인한다.

**예시: 알림 서비스 테스트**

```kotlin
// SUT: NotificationService
// Dependency: EmailClient (Interface)
interface EmailClient {
    fun send(to: String, subject: String, body: String)
}

class NotificationService(private val emailClient: EmailClient) {
    fun sendWelcomeEmail(email: String) {
        // ... 복잡한 비즈니스 로직 ...
        emailClient.send(
            to = email, 
            subject = "환영합니다!", 
            body = "TDD 마스터의 세계에 오신 것을 환영합니다."
        )
    }
}

// Test Code
class NotificationServiceTest : BehaviorSpec({
    given("NotificationService와 Mock EmailClient가 주어졌을 때") {
        // 1. mockk: EmailClient의 목 객체를 생성
        val mockEmailClient = mockk<EmailClient>()
        val notificationService = NotificationService(mockEmailClient)
        
        // 2. every-returns: 목 객체의 send 메소드는 아무것도 하지 않도록 정의 (Unit을 반환)
        every { mockEmailClient.send(any(), any(), any()) } returns Unit

        `when`("환영 이메일을 보내면") {
            notificationService.sendWelcomeEmail("test@example.com")
            
            then("EmailClient의 send 메소드를 정확한 인자와 함께 호출해야 한다") {
                // 3. verify: 행위를 검증
                verify(exactly = 1) {
                    mockEmailClient.send(
                        to = "test@example.com",
                        subject = "환영합니다!",
                        body = any() // body 내용은 복잡할 수 있으니 any()로 처리
                    )
                }
            }
        }
    }
})
```

### **단순 Mock을 넘어서는 고급 기능들**

MockK의 진가는 단순한 Mocking을 넘어, 복잡한 시나리오를 다룰 수 있는 고급 기능에서 드러난다.

#### **Spy: 실제 객체의 일부 행동만 바꾸기**

때로는 실제 객체의 대부분 로직은 그대로 사용하면서, 특정 메소드 하나만 테스트를 위해 다른 결과를 반환하도록 만들고 싶을 때가 있다. 이때 \*\*`spyk<T>()`\*\*를 사용한다. Spy는 실제 객체를 감싸는 프록시처럼 동작한다.

```kotlin
class RealEmailClient : EmailClient {
    override fun send(to: String, subject: String, body: String) {
        // 실제 이메일을 보내는 무거운 로직...
        throw IllegalStateException("실제 이메일 발송은 안돼!")
    }
    fun connect() { println("외부 SMTP 서버에 연결 성공") }
}

// ... 테스트 코드 ...
val spyEmailClient = spyk(RealEmailClient())
every { spyEmailClient.send(any(), any(), any()) } returns Unit // send 메소드만 가짜로 대체

spyEmailClient.connect() // connect()는 실제 객체의 메소드가 호출됨 -> "외부 SMTP 서버에 연결 성공" 출력
spyEmailClient.send("a", "b", "c") // send()는 우리가 정의한 대로 동작함 (예외 발생 안 함)
```

#### **Capturing: 목에 전달된 인자 엿보기**

메소드에 전달되는 인자가 단순한 값이 아니라 복잡한 객체일 때, 그 객체의 내부 프로퍼티 값이 올바르게 설정되었는지 검증하고 싶을 때가 있다. 이때 \*\*`slot()`\*\*과 \*\*`capture()`\*\*를 사용한다.

```kotlin
// ... 이전 테스트에서 이어서 ...
val subjectSlot = slot<String>()

verify(exactly = 1) {
    mockEmailClient.send(
        to = "test@example.com",
        subject = capture(subjectSlot), // subject 인자를 '캡처'
        body = any()
    )
}

// 캡처된 값 검증
val capturedSubject = subjectSlot.captured
capturedSubject shouldStartWith "환영"
capturedSubject.length shouldBe 5
```

#### **Hierarchical Stubbing: 연쇄 호출 한번에 정의하기**

`user.address.city`와 같이 객체의 연쇄적인 메소드 호출 결과를 Mocking 해야 할 때가 있다. MockK는 이를 매우 우아하게 처리한다.

```kotlin
val mockUser = mockk<User>()
// user.getAddress().getCity() 호출 시 "Seoul"을 반환하도록 한번에 정의
every { mockUser.getAddress().getCity() } returns "Seoul" 
```

MockK는 이 외에도 코루틴 지원, `relaxed` 목 등 코틀린 개발자를 위한 수많은 편의 기능을 제공한다. 이 강력한 도구를 통해 우리는 어떤 복잡한 의존성을 가진 객체라도 두려움 없이 테스트의 통제 하에 둘 수 있다. 이제 우리는 테스트의 구조(Kotest)와 내용(MockK)을 모두 다룰 수 있는 완벽한 준비를 마쳤다.