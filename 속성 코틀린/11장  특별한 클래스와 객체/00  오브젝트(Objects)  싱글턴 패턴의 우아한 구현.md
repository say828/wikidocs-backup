### 오브젝트(Objects): 싱글턴 패턴의 우아한 구현

**싱글턴(Singleton) 패턴**은 특정 클래스의 인스턴스가 애플리케이션 전체에 **오직 하나만** 존재하도록 보장하고, 이 인스턴스에 대한 전역적인 접근점을 제공하는 매우 흔한 디자인 패턴입니다. 데이터베이스 연결 관리자, 앱 환경 설정 객체 등 공유 자원을 다룰 때 주로 사용됩니다.

자바에서는 이러한 싱글턴을 구현하기 위해 private 생성자, 정적(static) 필드, `getInstance()` 메서드 등 복잡하고 장황한 코드를 작성해야 했으며, 스레드 안전성까지 직접 고려해야 하는 번거로움이 있었습니다.

#### `object` 키워드의 혁신

코틀린은 `object`라는 키워드 하나만으로 이 모든 복잡성을 해결합니다. `object`로 선언된 객체는 **그 선언과 동시에 단 하나의 인스턴스가 생성**되며, 이후 프로그램 어디에서든 그 이름을 통해 유일한 인스턴스에 접근할 수 있습니다.

**비유:** `object`는 세상에 단 하나뿐인 '남산타워' 🗼와 같습니다. 우리는 '새로운 남산타워'를 만들지 않습니다. 그저 '남산타워'라고 부르며 유일한 그 존재를 참조할 뿐입니다.

```kotlin
// AppConfig라는 이름의 싱글턴 객체 선언
object AppConfig {
    val serverUrl = "https://api.my-app.com"
    val maxConnections = 10

    fun printConfigInfo() {
        println("Server: $serverUrl, Max Connections: $maxConnections")
    }
}
```

`object`를 사용하는 방법은 매우 간단합니다. 클래스 이름처럼 보이는 `AppConfig`를 통해 프로퍼티와 메서드에 직접 접근하면 됩니다. 별도의 `getInstance()` 메서드 호출은 필요 없습니다.

```kotlin
fun main() {
    println("Connecting to ${AppConfig.serverUrl}")
    AppConfig.printConfigInfo()

    // AppConfig.serverUrl = "..." // val이므로 변경 불가

    // val config1 = AppConfig
    // val config2 = AppConfig
    // println(config1 === config2) // 출력: true (두 변수는 완전히 동일한 단일 인스턴스를 가리킴)
}
```

##### `object`의 특징

  * **지연 초기화와 스레드 안전성:** `object`는 해당 객체에 **처음 접근하는 순간**에 단 한 번만 초기화되며, 이 과정은 **스레드로부터 안전**하도록 코틀린 런타임이 보장해 줍니다.
  * **클래스처럼 행동:** `object`는 클래스를 상속받거나 인터페이스를 구현할 수 있습니다.
    ```kotlin
    interface EventListener { fun onEvent() }

    object GlobalEventHandler : EventListener {
        override fun onEvent() {
            println("Global event occurred!")
        }
    }
    ```

#### 객체 표현식 (Object Expression): 이름 없는 객체

`object` 키워드는 이름 없이 일회성으로 사용할 객체를 만들 때도 사용됩니다. 이를 **객체 표현식**이라고 하며, 자바의 \*\*익명 내부 클래스(Anonymous Inner Class)\*\*를 대체합니다.

주로 인터페이스나 추상 클래스의 인스턴스가 필요한 곳에서 즉석으로 구현체를 만들어 전달할 때 유용합니다.

```kotlin
// window.addMouseListener( new MouseAdapter() { ... } ) // Java의 익명 내부 클래스
window.addMouseListener(object : MouseAdapter() { // Kotlin의 객체 표현식
    override fun mouseClicked(e: MouseEvent) {
        // 클릭 이벤트 처리
    }

    override fun mouseEntered(e: MouseEvent) {
        // 마우스 진입 이벤트 처리
    }
})
```

`object` 키워드는 흔히 사용되는 싱글턴 패턴을 언어 차원에서 간결하고 안전하게 지원함으로써, 개발자가 보일러플레이트 코드 대신 비즈니스 로직에 더 집중할 수 있도록 돕는 코틀린의 실용주의 철학을 보여주는 대표적인 기능입니다.