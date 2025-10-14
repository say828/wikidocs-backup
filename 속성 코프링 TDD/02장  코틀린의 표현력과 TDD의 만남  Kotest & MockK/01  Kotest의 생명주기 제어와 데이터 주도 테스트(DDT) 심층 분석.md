## 01\. Kotest의 생명주기 제어와 데이터 주도 테스트(DDT) 심층 분석

Kotest의 진정한 힘은 서술적인 문법을 넘어, 테스트의 실행 흐름을 정밀하게 제어하고 반복적인 검증을 효율적으로 자동화하는 기능에서 드러난다. 좋은 테스트는 **FIRST** 원칙에 따라 독립적(Independent)이고 반복 가능(Repeatable)해야 한다. 이를 보장하기 위해 우리는 종종 테스트 실행 전후로 특정 준비(setup)나 정리(teardown) 작업이 필요하다. 바로 이 지점에서 Kotest의 **생명주기(Lifecycle) 제어 기능**이 빛을 발한다.

### **테스트의 흐름을 지휘하라: 생명주기 콜백**

Kotest는 테스트 스위트(Spec) 전체부터 개별 테스트 케이스에 이르기까지, 다양한 시점에 원하는 코드를 실행할 수 있는 콜백(Callback) 함수를 제공한다. 이를 통해 우리는 언제나 깨끗하고 예측 가능한 환경에서 테스트를 실행할 수 있다.

[도표: Kotest의 주요 생명주기 콜백]

| 콜백 함수 | 실행 시점 | 주요 용도 |
| :--- | :--- | :--- |
| `beforeSpec` | Spec(테스트 클래스)의 모든 테스트 실행 **전** 단 한 번 | DB 커넥션 풀 초기화, Testcontainers 시작 등 무거운 초기 설정 |
| `afterSpec` | Spec의 모든 테스트 실행 **후** 단 한 번 | DB 커넥션 풀 종료, Testcontainers 종료 등 전체 리소스 정리 |
| `beforeTest` | **각각의** 테스트 케이스(`given`, `when`, `then` 등) 실행 **전** | 테스트 데이터 삽입, Mock 객체 초기화 등 매번 독립적인 환경 구축 |
| `afterTest` | **각각의** 테스트 케이스 실행 **후** | 테스트 데이터 삭제, Mock 객체 상태 리셋 등 테스트 간 영향 격리 |

**실제 사용 예시: `BehaviorSpec`에서의 생명주기 제어**

```kotlin
import io.kotest.core.spec.style.BehaviorSpec

class DatabaseServiceTest : BehaviorSpec({
    // 1. Spec 전체에 대한 설정
    beforeSpec {
        println("== 데이터베이스 연결 풀을 초기화합니다. (단 한 번 실행) ==")
        // DatabaseConnection.initialize()
    }

    afterSpec {
        println("== 데이터베이스 연결 풀을 종료합니다. (단 한 번 실행) ==")
        // DatabaseConnection.shutdown()
    }

    // 2. 각 테스트 케이스에 대한 설정
    beforeTest {
        println("-- 테스트 데이터를 정리하고 삽입합니다. (매번 실행) --")
        // userRepository.deleteAll()
        // userRepository.save(User(name = "test-user"))
    }
    
    afterTest {
        println("-- 테스트 데이터를 정리합니다. (매번 실행) --")
        // userRepository.deleteAll()
    }

    given("사용자 데이터가 준비된 상태에서") {
        `when`("특정 사용자를 조회하면") {
            // val user = userService.findByName("test-user")
            then("해당 사용자의 정보가 반환되어야 한다") {
                // ... 검증 로직 ...
            }
        }
        `when`("존재하지 않는 사용자를 조회하면") {
            // val exception = shouldThrow<UserNotFoundException> { ... }
            then("예외가 발생해야 한다") {
                // ... 검증 로직 ...
            }
        }
    }
})
```

이러한 생명주기 제어를 통해 우리는 테스트 코드의 중복을 제거하고, 각 테스트가 서로에게 영향을 주지 않는 완벽한 격리 환경을 보장할 수 있다.

### **반복을 제거하는 기술: 데이터 주도 테스트 (Data-Driven Testing)**

같은 로직을 여러 개의 다른 입력 값으로 테스트해야 하는 경우가 있다. 예를 들어, 이메일 주소의 유효성을 검사하는 함수를 테스트한다면 수많은 정상 케이스와 비정상 케이스를 모두 검증해야 한다. 이때 각각의 케이스를 별도의 `then` 블록으로 만드는 것은 엄청난 코드 중복을 유발한다.

데이터 주도 테스트(DDT)는 이러한 문제를 우아하게 해결한다. 테스트 로직은 한 번만 작성하고, 테스트에 사용할 데이터만 테이블 형태로 제공하면 Kotest가 각 데이터를 순회하며 자동으로 테스트를 실행해준다.

**예시: 이메일 유효성 검사 DDT**

```kotlin
import io.kotest.core.spec.style.FunSpec
import io.kotest.datatest.withData

class EmailValidatorTest : FunSpec({
    context("이메일 주소 유효성 검사") {
        // withData에 테스트할 데이터 목록을 전달한다.
        withData(
            // (입력값, 기대 결과) 형태의 데이터
            "test@example.com" to true,
            "test.123@company.co.kr" to true,
            "invalid-email" to false,
            "test@.com" to false,
            "@example.com" to false
        ) { (email, expected) ->
            // 각 데이터에 대해 이 블록이 반복 실행된다.
            val isValid = EmailValidator.isValid(email)
            isValid shouldBe expected
        }
    }
})
```

`withData`를 사용함으로써, 우리는 단 몇 줄의 코드로 5개의 다른 테스트 케이스를 명확하고 간결하게 정의했다. 새로운 테스트 케이스를 추가하고 싶다면, 그저 데이터 목록에 한 줄을 추가하면 그만이다. 이는 테스트의 유지보수성을 극적으로 향상시킨다.

Kotest가 제공하는 생명주기 제어와 데이터 주도 테스트는 단순한 편의 기능을 넘어선다. 이 기능들은 **FIRST** 원칙을 지키며, 우리의 테스트 코드를 깨끗하고(Clean), 효율적이며(Efficient), 확장 가능한(Scalable) 소프트웨어 자산으로 만들어주는 핵심적인 도구다.