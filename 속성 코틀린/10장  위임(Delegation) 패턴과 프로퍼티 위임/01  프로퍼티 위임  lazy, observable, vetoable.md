### 프로퍼티 위임: lazy, observable, vetoable

클래스 위임이 인터페이스의 구현을 다른 객체에게 맡기는 것이었다면, \*\*프로퍼티 위임(Property Delegation)\*\*은 프로퍼티의 접근자(getter/setter) 로직을 다른 객체에게 맡기는 기술입니다.

프로퍼티는 단순히 값을 저장하는 것 외에도, 특정 패턴을 가지는 경우가 많습니다.

  * 처음 접근할 때 딱 한 번만 값을 계산하고 싶을 때
  * 값이 변경될 때마다 특정 동작(로깅, UI 갱신 등)을 수행하고 싶을 때
  * 새로운 값이 특정 조건을 만족할 때만 변경을 허용하고 싶을 때

이러한 공통 로직들을 매번 getter/setter에 직접 구현하는 대신, 재사용 가능한 **대리인(delegate) 객체**에게 위임하여 코드를 더 깔끔하고 모듈화되도록 만들 수 있습니다.

**기본 구조: `val/var <프로퍼티 이름>: <타입> by <대리인 객체>`**

코틀린 표준 라이브러리는 가장 흔하게 사용되는 세 가지 프로퍼티 대리인을 기본으로 제공합니다.

-----

#### 1\. `lazy`: 최초 사용 시 초기화

`by lazy`는 7장에서 지연 초기화 전략으로 이미 만났습니다. 이는 프로퍼티 위임의 가장 대표적인 예시입니다. `lazy`는 **읽기 전용(`val`)** 프로퍼티에 사용되며, 해당 프로퍼티에 **처음 접근하는 순간**에 단 한 번만 람다 블록을 실행하여 그 결과값을 프로퍼티의 값으로 확정합니다.

초기화 비용이 매우 큰 객체나, 사용할 수도 있고 안 할 수도 있는 프로퍼티에 적용하면 애플리케이션의 시작 성능을 향상시키는 데 큰 도움이 됩니다.

💡 **비유:** `lazy`는 꼭 필요할 때까지 일을 미루는 현명한 게으름뱅이와 같습니다.

```kotlin
class UserProfile(userId: String) {
    // profilePhoto 프로퍼티는 실제로 접근되기 전까지는 로딩되지 않습니다.
    val profilePhoto: ByteArray by lazy {
        println("[${userId}]의 프로필 사진을 로딩합니다... (이 메시지는 한 번만 보입니다)")
        // 네트워크나 파일 시스템에서 이미지를 읽어오는 무거운 작업
        loadImageFromServer(userId)
    }
}

fun main() {
    val user = UserProfile("user123")
    println("UserProfile 객체 생성 완료.")
    
    println("첫 번째 사진 접근:")
    user.profilePhoto // 이 순간 lazy 블록 실행
    
    println("\n두 번째 사진 접근:")
    user.profilePhoto // 이미 캐시된 값이므로 lazy 블록 실행 안 됨
}
```

-----

#### 2\. `observable`: 변경 감시

`Delegates.observable()`은 **`var`** 프로퍼티에 사용되며, 프로퍼티의 값이 **변경된 후에** 특정 동작을 수행하도록 할 수 있습니다.

`observable`은 초기값과 람다를 인자로 받습니다. 이 람다는 프로퍼티 값이 변경될 때마다 호출되며, 람다 안에서는 프로퍼티 정보, 이전 값(old value), 그리고 새로운 값(new value)에 접근할 수 있습니다.

📹 **비유:** `observable`은 프로퍼티에 설치된 CCTV와 같습니다. 값이 바뀔 때마다 변경 전후의 모습을 촬영하여 보고해 줍니다.

```kotlin
import kotlin.properties.Delegates

class User {
    // name 프로퍼티의 변경을 감시합니다.
    var name: String by Delegates.observable("<이름 없음>") { prop, oldValue, newValue ->
        println("${prop.name} 프로퍼티가 '${oldValue}'에서 '${newValue}'(으)로 변경되었습니다.")
    }
}

fun main() {
    val user = User()
    user.name = "Alice"
    user.name = "Bob"
}
```

**실행 결과:**

```
name 프로퍼티가 '<이름 없음>'에서 'Alice'(으)로 변경되었습니다.
name 프로퍼티가 'Alice'에서 'Bob'(으)로 변경되었습니다.
```

-----

#### 3\. `vetoable`: 변경 거부권

`Delegates.vetoable()`은 `observable`과 유사하지만, 프로퍼티 값이 **변경되기 전에** 람다를 호출하여 **변경을 허용할지 말지 결정**할 수 있는 거부권을 가집니다.

람다는 `Boolean` 값을 반환해야 하는데, `true`를 반환하면 변경이 승인되고, `false`를 반환하면 변경이 거부되어 프로퍼티 값은 이전 상태를 유지합니다.

🕴️ **비유:** `vetoable`은 엄격한 문지기와 같습니다. 새로운 값이 들어오려고 할 때마다 자격(람다 조건)을 심사하여, 통과할 때만 들여보내 줍니다.

```kotlin
import kotlin.properties.Delegates

class PositiveNumber {
    // 0 이상의 값만 허용하는 프로퍼티
    var value: Int by Delegates.vetoable(0) { prop, oldValue, newValue ->
        newValue >= 0 // 이 조건이 true일 때만 값이 변경됨
    }
}

fun main() {
    val number = PositiveNumber()
    println(number.value) // 0
    
    number.value = 10
    println(number.value) // 10 (변경 승인)
    
    number.value = -5
    println(number.value) // 10 (변경 거부, 이전 값 유지)
}
```

프로퍼티 위임은 `lazy`, `observable`, `vetoable`처럼 자주 사용되는 프로퍼티 로직을 재사용 가능한 객체로 캡슐화하여, 클래스의 본문은 핵심적인 비즈니스 로직에만 집중할 수 있도록 도와주는 강력한 패턴입니다.