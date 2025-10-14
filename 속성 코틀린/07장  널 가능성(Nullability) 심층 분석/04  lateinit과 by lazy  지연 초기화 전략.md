### lateinit과 by lazy: 지연 초기화 전략

코틀린의 엄격한 널 안정성 규칙은 Non-null 타입의 프로퍼티는 반드시 선언과 동시에 또는 생성자(`init` 블록 포함) 내에서 초기화되어야 한다고 강제합니다. 하지만 실제 개발에서는 객체가 생성된 이후에 프로퍼티 값이 정해지는 경우가 종종 있습니다.

  * 안드로이드 개발에서 `Activity`의 UI 컴포넌트는 `onCreate()` 메서드 안에서 초기화됩니다.
  * 의존성 주입(Dependency Injection) 프레임워크는 객체 생성 후에 의존성을 주입합니다.
  * 초기화 과정이 매우 비용이 커서, 실제로 그 프로퍼티가 처음 사용되는 시점에 초기화하고 싶을 때가 있습니다.

이러한 상황에서 무작정 프로퍼티를 `String? = null` 과 같이 Nullable로 만들고 `!!` 연산자를 남발하는 것은 널 안정성의 이점을 스스로 포기하는 것입니다. 코틀린은 이런 딜레마를 해결하기 위해 \*\*`lateinit`\*\*과 \*\*`by lazy`\*\*라는 두 가지 지연 초기화 전략을 제공합니다.

-----

#### `lateinit`: 나중에 반드시 초기화하겠다는 약속

`lateinit`은 \*\*'late initialize(나중에 초기화하다)'\*\*의 줄임말로, Non-null 타입의 `var` 프로퍼티에 사용됩니다. 이는 개발자가 컴파일러에게 다음과 같이 약속하는 것과 같습니다.
**"이 프로퍼티는 지금 당장 초기화할 수 없지만, 내가 약속하건대 이 프로퍼티를 처음 사용하기 전까지는 반드시 초기화할 거야. 그러니 일단 컴파일은 통과시켜 줘."** 🤝

##### `lateinit`의 규칙과 특징

  * `var` 프로퍼티에만 사용할 수 있습니다 (`val`은 불가).
  * `Int`, `Boolean` 등 원시 타입(Primitive Type)에는 사용할 수 없습니다.
  * Nullable 타입에는 사용할 수 없습니다 (`lateinit var name: String?`는 의미가 없으므로 금지).

<!-- end list -->

```kotlin
// 안드로이드 Activity의 간단한 예시
class MyActivity {
    // 아직 초기화할 수 없지만, null이 될 수 없는 TextView 프로퍼티
    private lateinit var textView: TextView
    private lateinit var apiService: ApiService

    // Activity가 생성된 후 onCreate에서 실제 값 할당
    fun onCreate() {
        textView = findViewById(R.id.my_text_view)
        apiService = ApiService.create()
        println("초기화 완료!")
    }

    fun loadData() {
        // 이제 안전하게 프로퍼티 사용 가능
        val data = apiService.getData()
        textView.text = data
    }
}
```

**주의할 점:** 만약 개발자가 약속을 어기고 `lateinit` 프로퍼티를 초기화하기 전에 접근하면, `UninitializedPropertyAccessException`이라는 런타임 예외가 발생합니다. 이는 NPE보다는 원인을 명확히 알려주지만, 여전히 프로그램 중단을 유발하므로 `lateinit`을 사용한 프로퍼티는 라이프사이클 내에서 반드시 초기화가 보장되어야 합니다.

-----

#### `by lazy`: 처음 사용할 때 단 한 번 초기화되는 값

`by lazy`는 **읽기 전용(`val`) 프로퍼티**를 위한 지연 초기화 기법입니다. 프로퍼티 선언과 함께 초기화 로직을 람다로 전달하면, 해당 프로퍼티는 **객체가 생성될 때가 아닌, 코드상에서 처음으로 접근되는 시점**에 단 한 번만 초기화됩니다.

**비유:** `by lazy`는 잘 사용하지 않는 방의 '동작 감지 센서 등 💡'과 같습니다. 방에 들어갈 때(프로퍼티에 처음 접근할 때) 불이 켜지고(초기화 실행), 그 이후부터는 계속 켜진 상태를 유지합니다(초기화된 값 재사용).

##### `by lazy`의 특징

  * `val` 프로퍼티에만 사용할 수 있습니다 (`var`은 불가).
  * 초기화 블록(`{...}`)이 최초 호출 시 단 한 번만 실행됩니다.
  * 초기화된 값은 캐시되어 이후에는 계속 그 값을 반환합니다.
  * 기본적으로 스레드로부터 안전(thread-safe)하게 동작합니다.

<!-- end list -->

```kotlin
class UserSession(private val userId: String) {
    // userProfile 프로퍼티는 처음 접근될 때 데이터베이스 조회를 통해 초기화됩니다.
    val userProfile: UserProfile by lazy {
        println("데이터베이스에서 사용자 프로필을 로딩합니다...")
        // 데이터베이스 조회나 네트워크 요청 등 비용이 큰 작업
        loadProfileFromDB(userId)
    }
}

fun main() {
    val session = UserSession("user123")
    println("세션 객체가 생성되었습니다.") // 이 시점에는 아직 DB 조회가 일어나지 않음

    println("사용자 이름에 처음 접근합니다...")
    println(session.userProfile.name) // 이 순간 lazy 블록이 실행됨
    
    println("\n사용자 이메일에 다시 접근합니다...")
    println(session.userProfile.email) // 이미 초기화되었으므로 lazy 블록이 실행되지 않음
}
```

**실행 결과:**

```
세션 객체가 생성되었습니다.
사용자 이름에 처음 접근합니다...
데이터베이스에서 사용자 프로필을 로딩합니다...
Alice
사용자 이메일에 다시 접근합니다...
alice@example.com
```

-----

#### `lateinit` vs. `by lazy`: 언제 무엇을 쓸까?

| 구분 | `lateinit var` | `val ... by lazy` |
| :--- | :--- | :--- |
| **대상** | `var` (변경 가능 프로퍼티) | `val` (읽기 전용 프로퍼티) |
| **초기화 시점** | 개발자가 코드 내에서 **직접** 할당 | **최초 접근 시** 자동으로 할당 |
| **주요 용도** | 의존성 주입, 안드로이드 생명주기 | 초기화 비용이 큰 프로퍼티, 싱글턴 |
| **스레드 안전성** | 보장 안 됨 | 기본적으로 보장됨 |

`lateinit`과 `by lazy`는 Non-null 프로퍼티의 초기화 시점을 유연하게 제어하여, 코틀린의 널 안정성을 해치지 않으면서 실제 개발 환경의 다양한 요구사항에 대응할 수 있게 해주는 매우 실용적인 도구입니다.