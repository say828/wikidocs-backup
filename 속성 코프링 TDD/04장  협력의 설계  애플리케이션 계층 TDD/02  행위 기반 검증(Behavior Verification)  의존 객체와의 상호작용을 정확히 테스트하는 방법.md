## 02\. 행위 기반 검증(Behavior Verification): 의존 객체와의 상호작용을 정확히 테스트하는 방법

앞서 우리는 목(Mock) 객체의 목적이 SUT(테스트 대상 시스템)와 협력 객체 간의 '상호작용'을 검증하는 것이라고 배웠다. \*\*행위 기반 검증(Behavior Verification)\*\*은 바로 이 상호작용을 테스트하는 구체적인 기술을 의미한다. 애플리케이션 서비스처럼 자신의 로직은 거의 없이 여러 의존 객체들을 조율하는 '지휘자' 객체를 테스트할 때, 이 기법은 절대적인 힘을 발휘한다.

이번 절에서는 TDD 사이클을 통해, 행위 기반 검증이 어떻게 애플리케이션 서비스의 역할과 책임을 명확하게 정의하고 설계하도록 이끄는지 실제 예제와 함께 깊이 있게 탐구한다.

### **Use Case: "사용자 비활성화"**

새로운 요구사항이 들어왔다고 가정하자. "사용자를 비활성화하면, 해당 사용자의 상태를 `DEACTIVATED`로 변경하고, 비활성화 안내 이메일을 발송해야 한다."

이 유스케이스를 구현할 `DeactivateUserService`를 TDD로 개발해보자. 이 서비스는 `UserRepository`와 `EmailClient`라는 두 개의 협력 객체(의존성)를 가질 것이다.

#### **1단계: 실패하는 테스트 작성 (RED) - 상호작용을 먼저 정의하라**

런던파 TDD 스타일에서는, 구현 코드를 작성하기 전에 **SUT가 어떤 협력 객체의 어떤 메소드를 호출해야 하는지**부터 테스트로 명세한다.

```kotlin
class DeactivateUserServiceTest : BehaviorSpec({
    // Collaborators
    lateinit var userRepository: UserRepository
    lateinit var emailClient: EmailClient
    
    // System Under Test
    lateinit var deactivateUserService: DeactivateUserService

    // `beforeTest`를 사용해 매번 깨끗한 객체들로 테스트 환경을 구성한다.
    beforeTest {
        userRepository = mockk(relaxed = true)
        emailClient = mockk(relaxed = true)
        deactivateUserService = DeactivateUserService(userRepository, emailClient)
    }

    given("활성화된 사용자가 존재할 때") {
        val userId = 1L
        val userEmail = "test@example.com"
        val activeUser = User(id = userId, email = userEmail, status = UserStatus.ACTIVE)
        
        // Stub: 테스트 실행을 위해 userRepository가 activeUser를 반환하도록 설정
        every { userRepository.findById(userId) } returns activeUser

        `when`("해당 사용자를 비활성화하면") {
            deactivateUserService.deactivate(userId)

            then("사용자를 조회하고, 비활성화 상태로 저장하며, 안내 이메일을 발송해야 한다") {
                // Verify: 아직 구현되지 않은 SUT의 '행위'를 먼저 검증한다.
                verifyOrder {
                    // 1. findById가 먼저 호출되었는지 검증
                    userRepository.findById(userId)
                    
                    // 2. save가 그 다음에 호출되었는지 검증
                    userRepository.save(any()) 
                    
                    // 3. sendDeactivationEmail이 마지막으로 호출되었는지 검증
                    emailClient.sendDeactivationEmail(userEmail)
                }
            }
        }
    }
})
```

`verifyOrder`는 메소드들이 **정확한 순서대로** 호출되었는지까지 검증하는 강력한 기능이다. 이 테스트는 `DeactivateUserService`가 존재하지 않으므로 당연히 컴파일조차 되지 않는 **RED** 상태다.

#### **2단계: 테스트 통과 (GREEN) - 최소한의 코드로 구현**

이제 이 테스트를 통과시키기 위한 가장 간단한 코드를 작성한다.

```kotlin
class DeactivateUserService(
    private val userRepository: UserRepository,
    private val emailClient: EmailClient
) {
    fun deactivate(userId: Long) {
        // 1. 사용자 조회
        val user = userRepository.findById(userId) 
            ?: throw UserNotFoundException()

        // 2. 도메인 객체에 비활성화 책임 위임
        user.deactivate() // user.status = DEACTIVATED

        // 3. 변경된 사용자 저장
        userRepository.save(user)

        // 4. 이메일 발송
        emailClient.sendDeactivationEmail(user.email)
    }
}
```

이제 테스트는 통과하여 **GREEN** 상태가 된다. 우리는 `DeactivateUserService`가 올바른 순서로 협력 객체들과 상호작용한다는 것을 보장하는 안전망을 확보했다.

#### **3단계: 리팩토링 및 검증 강화 (REFACTOR)**

현재 테스트는 `userRepository.save(any())`를 통해 `save` 메소드가 '어떤 User 객체이든' 받아서 호출되었는지만을 검증한다. 하지만 우리는 `save` 메소드에 전달된 `User` 객체의 상태가 정말 `DEACTIVATED`로 변경되었는지 확인하고 싶다. **행위 기반 검증에 상태 기반 검증을 결합**하면 테스트를 훨씬 더 견고하게 만들 수 있다. 이때 MockK의 `slot`과 `capture`가 사용된다.

```kotlin
// ... 이전 테스트 코드의 then 블록 수정 ...
then("사용자를 조회하고, '비활성화된 상태로' 저장하며, 안내 이메일을 발송해야 한다") {
    val userSlot = slot<User>() // User 객체를 담을 '슬롯'을 생성

    verifyOrder {
        userRepository.findById(userId)
        userRepository.save(capture(userSlot)) // save 호출 시 전달된 User 객체를 슬롯에 캡처
        emailClient.sendDeactivationEmail(userEmail)
    }

    // 캡처된 객체의 '상태'를 검증한다.
    val capturedUser = userSlot.captured
    capturedUser.status shouldBe UserStatus.DEACTIVATED
}
```

이제 우리의 테스트는 서비스의 행위(메소드 호출 순서)와 그 행위의 결과(저장되는 객체의 최종 상태)를 모두 검증하는 훨씬 더 강력하고 의미 있는 테스트가 되었다.

행위 기반 검증은 애플리케이션 계층의 TDD를 위한 핵심 기술이다. 이 기법을 통해 우리는 느리고 복잡한 인프라에 대한 의존 없이, 오직 SUT와 협력 객체들의 순수한 상호작용에만 집중하여 유스케이스의 정확성을 보장할 수 있다. 이는 각 객체의 역할과 책임을 명확히 분리하고, 유연하며 유지보수하기 좋은 아키텍처를 설계하는 데 결정적인 역할을 한다.