### KMP의 철학과 아키텍처 이해

#### 철학: 핵심 로직은 공유하고, UI는 네이티브로

KMP의 가장 중요한 철학은 \*\*"공유할 수 있는 것만 공유한다"\*\*는 실용주의에 있습니다. 특히 사용자 인터페이스(UI)는 각 플랫폼의 생태계와 사용자 경험에 가장 깊이 관여된 부분이므로, 이를 억지로 공통화하기보다는 각 플랫폼의 네이티브 방식(Android는 Jetpack Compose, iOS는 SwiftUI)을 그대로 사용하는 것을 권장합니다.

대신, 다음과 같이 플랫폼에 독립적인 로직들을 공통 모듈에 작성하여 공유합니다.

  * **비즈니스 로직:** 앱의 핵심 규칙과 알고리즘
  * **데이터 모델:** `User`, `Product` 와 같이 앱에서 사용하는 데이터 구조
  * **데이터 소스:** 네트워크 API 호출, 데이터베이스 접근, 저장소(Repository) 패턴
  * **유틸리티:** 날짜/시간 처리, 포맷팅, 유효성 검사 등

**비유:** KMP는 여러 종류의 자동차(Android 앱, iOS 앱)에 장착할 수 있는 \*\*하나의 고성능 공유 엔진(공통 로직)\*\*을 만드는 것과 같습니다. 엔진은 동일하지만, 각 자동차의 외관 디자인과 운전석(UI)은 해당 자동차에 가장 최적화된 형태로 만들어 최상의 경험을 제공하는 것입니다.

#### 아키텍처: `expect`와 `actual`의 약속

그렇다면 공통 로직에서 파일 시스템 접근이나 기기 고유 ID 획득처럼 플랫폼마다 구현이 다른 기능은 어떻게 사용해야 할까요? 이때 KMP의 핵심적인 아키텍처 패턴인 \*\*`expect`와 `actual`\*\*이 등장합니다.

  * **`expect` (기대 선언): 공통 모듈의 약속**
    공통 모듈(`commonMain`)에서는 특정 기능에 대한 \*\*'기대'\*\*를 선언합니다. 이는 "이런 기능이 각 플랫폼에 존재할 것으로 기대한다"는 일종의 \*\*'추상적인 약속'\*\*과 같습니다. `expect`로 선언된 함수나 클래스는 본문을 가지지 않습니다.

    ```kotlin
    // shared/src/commonMain/kotlin/Platform.kt

    // 각 플랫폼이 이 함수를 구현해 줄 것을 '기대'합니다.
    expect fun getPlatformName(): String
    ```

  * **`actual` (실제 구현): 각 플랫폼의 화답**
    각 플랫폼별 소스셋(`androidMain`, `iosMain` 등)에서는 `expect`로 선언된 약속에 대한 \*\*'실제 구현'\*\*을 제공해야 합니다. 이때 `actual` 키워드를 사용하여 `expect` 선언과 짝을 이룸을 명시합니다.

    ```kotlin
    // shared/src/androidMain/kotlin/Platform.kt

    // commonMain의 expect 선언에 대한 '실제 구현'
    actual fun getPlatformName(): String {
        return "Android ${android.os.Build.VERSION.SDK_INT}"
    }
    ```

    ```kotlin
    // shared/src/iosMain/kotlin/Platform.kt

    import platform.UIKit.UIDevice

    // commonMain의 expect 선언에 대한 '실제 구현'
    actual fun getPlatformName(): String {
        return "${UIDevice.currentDevice.systemName()} ${UIDevice.currentDevice.systemVersion}"
    }
    ```

공통 로직에서는 `getPlatformName()` 함수를 마치 일반 함수처럼 호출하기만 하면 됩니다. 그러면 KMP 빌드 시스템이 코드를 컴파일하는 대상 플랫폼에 맞춰 올바른 `actual` 구현을 자동으로 연결해 줍니다.  이 `expect`/`actual` 메커니즘을 통해, 공통 코드는 플랫폼의 세부 사항을 전혀 몰라도 플랫폼별 기능을 안전하게 사용할 수 있습니다.

이처럼 KMP는 네이티브 플랫폼을 존중하는 철학과 `expect`/`actual`이라는 유연한 아키텍처를 통해, 코드 공유의 효율성과 네이티브 UI의 우수한 사용자 경험이라는 두 마리 토끼를 모두 잡는 현명한 길을 제시합니다.