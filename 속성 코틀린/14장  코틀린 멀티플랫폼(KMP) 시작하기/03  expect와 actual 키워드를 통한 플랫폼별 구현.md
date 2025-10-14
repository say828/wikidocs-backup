### expect와 actual 키워드를 통한 플랫폼별 구현

`expect`와 `actual` 키워드는 KMP의 철학을 코드로 구현하는 핵심 기술이자, 공통 모듈과 플랫폼별 모듈을 연결하는 가장 중요한 '다리'입니다. 이 메커니즘을 통해 공통 코드는 특정 플랫폼을 전혀 몰라도, 컴파일 시점에 각 플랫폼의 네이티브 기능과 안전하게 연결될 수 있습니다.

\*\*`expect`\*\*는 `commonMain`에서 "이러한 기능이 존재할 것으로 \*\*기대(expect)\*\*한다"고 선언하는 **청사진 또는 약속**입니다.
\*\*`actual`\*\*은 각 플랫폼 모듈(`androidMain`, `iosMain` 등)에서 "당신이 기대한 그 기능을 우리가 \*\*실제(actual)\*\*로 이렇게 구현했다"고 응답하는 **구현체**입니다.

-----

#### `expect`/`actual`의 규칙

  * **1대 N 관계:** 하나의 `expect` 선언이 있다면, 이 `commonMain` 모듈을 사용하는 **모든** 플랫폼 모듈은 반드시 해당 `expect`에 대한 `actual` 구현을 제공해야 합니다. (만약 `androidMain`, `iosMain`, `jsMain`을 타겟으로 한다면 세 곳 모두에 `actual` 구현이 필요합니다.)
  * **다양한 대상:** 이 메커니즘은 함수뿐만 아니라 클래스, 인터페이스, 객체(object), 프로퍼티 등 거의 모든 코틀린 선언에 적용할 수 있습니다.

-----

#### 1\. `expect` 함수 (기본 활용)

가장 간단한 사용법은 플랫폼별 상수값이나 간단한 기능을 함수로 가져오는 것입니다. 기기 고유의 UUID를 가져오는 기능을 예로 들어 봅시다.

**`commonMain` (공통 모듈)**

```kotlin
// "각 플랫폼은 UUID를 문자열로 반환하는 getUuid 함수를 구현해야 한다"고 기대
expect fun getUuid(): String

// 공통 뷰모델은 이 약속(expect)만 보고 코드를 작성
class CommonViewModel {
    fun generateDeviceId() = "KMP-DEVICE-${getUuid()}"
}
```

**`androidMain` (안드로이드 구현)**

```kotlin
import android.provider.Settings

// getUuid()에 대한 실제 구현을 제공
actual fun getUuid(): String {
    // 안드로이드 SDK의 Settings.Secure API를 사용
    return Settings.Secure.getString(ApplicationContextHolder.context.contentResolver, Settings.Secure.ANDROID_ID)
}
```

**`iosMain` (iOS 구현)**

```kotlin
import platform.UIKit.UIDevice

// getUuid()에 대한 실제 구현을 제공
actual fun getUuid(): String {
    // iOS의 UIDevice API를 사용
    return UIDevice.currentDevice.identifierForVendor?.UUIDString ?: "unknown-uuid"
}
```

이제 `CommonViewModel`의 `generateDeviceId()` 함수는 자신이 안드로이드에서 실행되는지 iOS에서 실행되는지 전혀 몰라도, 각 플랫폼에 맞는 `actual` 함수를 자동으로 호출하여 올바른 UUID를 가져올 수 있습니다.

#### 2\. `expect` 클래스 및 팩토리 (고급 활용)

더 나아가, 플랫폼별로 구현 방식이 완전히 다른 복잡한 기능(예: 데이터베이스, Key-Value 저장소, 네트워킹 엔진)도 이 패턴으로 추상화할 수 있습니다.

가장 일반적인 패턴은 `commonMain`에 공통 **인터페이스**를 정의하고, 이 인터페이스의 실제 구현체를 생성하는 **팩토리**를 `expect`로 선언하는 것입니다. 간단한 Key-Value 저장소를 예로 들어 보겠습니다.

**`commonMain` (공통 모듈: 규약 정의)**

```kotlin
// 1. 공통으로 사용할 인터페이스를 정의
interface KeyValueStore {
    fun put(key: String, value: String)
    fun get(key: String): String?
}

// 2. 이 인터페이스의 인스턴스를 생성해 줄 팩토리를 '기대'
expect object StoreFactory {
    fun createStore(): KeyValueStore
}
```

**`androidMain` (안드로이드 구현: SharedPreferences)**

```kotlin
import android.content.Context
// ... (Context를 받아오는 로직이 필요하다고 가정)

// 3. Android용 실제 팩토리 구현
actual object StoreFactory {
    actual fun createStore(): KeyValueStore {
        return AndroidStore(ApplicationContextHolder.context)
    }
}

// 4. SharedPreferences를 사용한 실제 클래스 구현
private class AndroidStore(context: Context) : KeyValueStore {
    private val prefs = context.getSharedPreferences("kmp_prefs", Context.MODE_PRIVATE)

    override fun put(key: String, value: String) = prefs.edit().putString(key, value).apply()
    override fun get(key: String): String? = prefs.getString(key, null)
}
```

**`iosMain` (iOS 구현: NSUserDefaults)**

```kotlin
import platform.Foundation.NSUserDefaults

// 3. iOS용 실제 팩토리 구현
actual object StoreFactory {
    actual fun createStore(): KeyValueStore {
        return IosStore()
    }
}

// 4. NSUserDefaults를 사용한 실제 클래스 구현
private class IosStore : KeyValueStore {
    private val defaults = NSUserDefaults.standardUserDefaults

    override fun put(key: String, value: String) = defaults.setObject(value, key)
    override fun get(key: String): String? = defaults.stringForKey(key) as String?
}
```

이제 `commonMain`의 코드는 `StoreFactory.createStore()`를 호출하기만 하면, 안드로이드에서는 `SharedPreferences`가, iOS에서는 `NSUserDefaults`가 동작하는 실제 구현체를 얻게 됩니다. `expect`/`actual`은 이처럼 순수한 공통 코드와 복잡한 네이티브 구현 사이의 간극을 메우는 완벽하고 타입 안전한 접착제 역할을 수행합니다.