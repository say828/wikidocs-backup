### common, jvm, js, native 소스 세트 구조

코틀린 멀티플랫폼(KMP) 프로젝트의 심장부에는 **소스 세트(Source Sets)**라는 독특하고 강력한 디렉터리 구조가 자리 잡고 있습니다. 소스 세트는 KMP의 철학인 '로직 공유'를 물리적으로 구현하는 방식이며, 어떤 코드가 모든 플랫폼에 공통으로 쓰이고 어떤 코드가 특정 플랫폼에서만 쓰이는지를 Gradle 빌드 시스템에 알려주는 역할을 합니다.

이 구조를 이해하는 것은 KMP 프로젝트에서 '어디에 어떤 코드를 작성해야 하는지'를 아는 첫걸음입니다.

---

#### 핵심 소스 세트: `commonMain`

**`commonMain`**은 KMP 프로젝트의 **심장이자 두뇌**입니다. 이 디렉터리에 작성된 코드는 **100% 순수 코틀린**으로, 프로젝트가 대상으로 하는 모든 플랫폼(Android, iOS, Desktop, Web 등)용으로 각각 컴파일되어 모두에게 공유됩니다.

* **역할:** 대부분의 공유 로직이 이곳에 위치합니다.
    * 데이터 클래스 (DTOs, Models)
    * 비즈니스 로직 (알고리즘, 유효성 검사)
    * 인터페이스 및 저장소(Repository) 선언
    * 네트워크 API 정의
    * 플랫폼별 구현을 요구하는 `expect` 선언
* **제약:** 이 코드는 순수 코틀린이어야 하므로, `android.content.Context`나 iOS의 `UIKit` 같은 특정 플랫폼의 API에 전혀 접근할 수 없습니다.

---

#### 플랫폼별 소스 세트: `androidMain`과 `iosMain`

`commonMain`이 모든 플랫폼의 공통분모라면, 플랫폼별 소스 세트는 각 플랫폼의 고유한 기능을 구현하는 '다리' 역할을 합니다.

##### `androidMain` (안드로이드 구현)
* **역할:** 오직 **Android 타겟**을 위해서만 컴파일되는 코틀린 코드입니다.
* **내용:**
    * `commonMain`에 선언된 `expect` 선언에 대한 **`actual` 구현**을 작성합니다. (예: 안드로이드 기기 UUID를 가져오는 `actual` 함수)
    * 안드로이드 SDK 및 라이브러리에 접근할 수 있습니다.
* **결과물:** 이 코드는 `commonMain` 코드와 합쳐져 표준 안드로이드 라이브러리(.aar 또는 .jar)로 컴파일됩니다.

##### `iosMain` (iOS 네이티브 구현)
* **역할:** **iOS 타겟**을 위해 Kotlin/Native 컴파일러로 처리되는 코틀린 코드입니다.
* **내용:**
    * 마찬가지로 `commonMain`의 `expect` 선언에 대한 `actual` 구현을 작성합니다. (예: iOS의 `UIDevice`를 사용하여 플랫폼 이름을 가져오는 `actual` 함수)
    * iOS/macOS의 네이티브 프레임워크(Foundation, UIKit 등)를 직접 임포트하고 호출할 수 있습니다.
* **결과물:** 이 코드는 `commonMain` 코드와 합쳐져 Xcode가 이해할 수 있는 네이티브 프레임워크(`.framework`)로 컴파일됩니다.



---

#### 소스 세트의 계층 구조와 의존성

소스 세트 간에는 명확한 의존성 계층이 존재합니다.

* **`androidMain`은 `commonMain`에 의존합니다.** (즉, `androidMain`의 코드는 `commonMain`의 코드를 알고 있으며 호출할 수 있습니다.)
* **`iosMain`은 `commonMain`에 의존합니다.** (마찬가지로 `iosMain`도 `commonMain`의 코드를 호출할 수 있습니다.)
* 하지만 **`commonMain`은 `androidMain`이나 `iosMain`의 존재를 전혀 알지 못합니다.** (오직 자신이 선언한 `expect`의 존재만 압니다.)

이러한 단방향 의존성 덕분에 `commonMain`은 어떠한 특정 플랫폼에도 종속되지 않는 순수성을 유지할 수 있으며, 이는 KMP 아키텍처의 핵심적인 강점입니다.

#### 더 넓은 확장: `jvm`, `js`, `native`

`androidMain`과 `iosMain`은 가장 일반적인 조합일 뿐, KMP는 더 많은 타겟을 지원합니다.
* **`jvmMain`:** 안드로이드뿐만 아니라, Ktor나 Spring Boot 같은 JVM 기반 **서버 백엔드**, 또는 **데스크톱(Compose for Desktop)** 애플리케이션을 위한 공통 JVM 코드를 담습니다. (`androidMain`은 `jvmMain`의 특별한 하위 분류로 볼 수 있습니다.)
* **`jsMain`:** 코틀린 코드를 **JavaScript로 컴파일**하여 웹 프론트엔드(React, Angular 등)에서 사용하기 위한 코드입니다.
* **`nativeMain`:** iOS, Linux, Windows, macOS 등 **모든 네이티브 타겟**의 공통 코드를 담습니다.

이 모든 구조의 정점에는 항상 `commonMain`이 존재하며, `commonMain`의 코드를 최대한 많이 작성하는 것이 KMP 프로젝트의 핵심 목표입니다.