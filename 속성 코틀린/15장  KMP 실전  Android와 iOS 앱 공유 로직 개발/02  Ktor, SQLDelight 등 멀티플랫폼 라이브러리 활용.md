### Ktor, SQLDelight 등 멀티플랫폼 라이브러리 활용

우리가 앞선 예제에서 `ApiClient`와 `User` 데이터 모델을 `commonMain`에 작성할 수 있었던 것은, 단순히 코틀린 언어의 기능 때문만이 아닙니다. 이는 Ktor와 kotlinx.serialization처럼 KMP의 철학을 완벽하게 지원하는 강력한 **멀티플랫폼 라이브러리 생태계**가 존재하기에 가능한 일이었습니다.

KMP 프로젝트의 성공은 핵심 로직을 얼마나 잘 공유하느냐에 달려있으며, 이는 곧 얼마나 검증된 멀티플랫폼 라이브러리를 잘 활용하느냐와 직결됩니다. 이 라이브러리들은 `expect`/`actual` 패턴을 내부적으로 사용하여, 우리가 플랫폼의 세부 사항을 신경 쓰지 않도록 복잡성을 완벽하게 추상화해 줍니다. KMP 데이터 스택을 구성하는 핵심 라이브러리들을 살펴보겠습니다.

#### Ktor (네트워킹)

**Ktor**는 JetBrains가 직접 만든 경량의 비동기 네트워킹 프레임워크입니다. KMP 프로젝트에서 사실상의 표준 HTTP 클라이언트로 사용됩니다.

우리가 `commonMain`에서 `HttpClient()`를 호출하기만 하면, Ktor는 의존성 설정을 기반으로 알아서 플랫폼별 실제 엔진을 주입합니다.
* **`build.gradle.kts` 설정:**
    * `commonMain`에는 `ktor-client-core` (공통 API)를 추가합니다.
    * `androidMain`에는 `ktor-client-android` (또는 `ktor-client-okhttp`)를 추가합니다. 이는 내부적으로 OkHttp를 사용하는 `actual` 구현체입니다.
    * `iosMain`에는 `ktor-client-darwin`을 추가합니다. 이는 iOS의 네이티브 `NSURLSession`을 사용하는 `actual` 구현체입니다.

개발자는 `commonMain`에서 Ktor의 공통 API(`client.get`, `client.post`)만을 사용하며, 실제 네트워크 요청은 각 플랫폼에서 가장 성능이 좋은 네이티브 방식으로 처리됩니다.

#### kotlinx.serialization (데이터 직렬화)

**`kotlinx.serialization`**은 코틀린 컴파일러 플러그인을 기반으로 동작하는 공식 직렬화 라이브러리입니다. 리플렉션(Reflection)에 의존하지 않고 컴파일 시점에 필요한 코드를 생성하므로, Android(R8/ProGuard) 환경이나 Kotlin/Native(iOS) 환경에서도 매우 빠르고 안정적으로 동작합니다.

앞서 본 `@Serializable` 어노테이션이 바로 이 라이브러리의 핵심이며, Ktor는 이 라이브러리와 완벽하게 통합되어 JSON 응답을 데이터 클래스로 자동 변환해 줍니다.

#### SQLDelight (데이터베이스 퍼시스턴스)

모든 앱은 네트워크 데이터 외에도 로컬 데이터를 저장할 공간이 필요합니다. Android에는 Room이 있지만 iOS에서는 사용할 수 없습니다. 이 문제를 해결하는 가장 강력한 KMP 라이브러리가 바로 **SQLDelight**입니다.

SQLDelight는 Room이나 Realm과 같은 ORM(객체 관계 매핑)이 아닙니다. SQLDelight의 철학은 독특합니다.
1.  **SQL이 곧 진리:** 개발자는 `.sq` 파일에 순수한 **표준 SQL 쿼리**(CREATE TABLE, INSERT, SELECT 등)를 직접 작성합니다.
2.  **컴파일 타임 코드 생성:** SQLDelight 컴파일러가 이 SQL 쿼리문들을 분석하여, 100% 코틀린으로 작성된 **타입 안전(type-safe)한 함수와 데이터 모델**을 자동으로 생성해 줍니다. (예: `getAllUsers()` 함수, `User` 데이터 클래스)
3.  **드라이버 추상화:** 실제 데이터베이스 엔진(SQLite)에 접근하는 드라이버는 `expect`/`actual` 패턴으로 추상화합니다.
    * `androidMain`에서는 `AndroidSqliteDriver`를 주입합니다.
    * `iosMain`에서는 `NativeSqliteDriver`를 주입합니다.

개발자는 `commonMain`에서 SQLDelight가 생성해준 타입 안전한 코틀린 함수(`db.userQueries.getAllUsers()`)만 호출하면, 각 플랫폼의 네이티브 SQLite 데이터베이스에 데이터가 안전하게 저장되고 조회됩니다.

#### kotlinx.coroutines (동시성)

이 모든 것을 하나로 묶어주는 접착제가 바로 **코루틴**입니다. `kotlinx-coroutines-core`는 완벽한 멀티플랫폼 라이브러리입니다. 우리가 작성한 `suspend fun` 함수는 Ktor의 비동기 네트워크 호출과 SQLDelight의 비동기 DB 접근을 `commonMain` 안에서 동일한 문법으로 일관되게 처리할 수 있도록 해줍니다.

이처럼 KMP 생태계는 네트워킹(Ktor), 직렬화(Serialization), 데이터베이스(SQLDelight), 동시성(Coroutines)이라는 앱 개발의 4대 핵심 요소를 완벽하게 지원합니다. 이 검증된 스택을 활용함으로써 우리는 앱의 핵심 두뇌 전체를 플랫폼에 종속되지 않는 순수한 공통 코드로 작성할 수 있습니다.