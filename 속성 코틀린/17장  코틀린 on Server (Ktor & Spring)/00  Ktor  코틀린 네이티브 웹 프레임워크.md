### Ktor: 코틀린 네이티브 웹 프레임워크

\*\*Ktor(케이토)\*\*는 코틀린을 만든 JetBrains가 직접 개발한, **코루틴 네이티브** 웹 프레임워크입니다. Ktor는 경량이며, 유연하고, 오직 코틀린만으로 비동기 웹 애플리케이션을 밑바닥부터 구축할 수 있도록 설계되었습니다.

#### Ktor의 철학

1.  **경량 및 비주도성(Lightweight & Unopinionated):** Ktor는 Spring Boot처럼 거대한 아키텍처를 강요하지 않습니다. 개발자는 최소한의 서버 엔진(Netty, Jetty 등)으로 시작하여, 필요한 기능만 **플러그인(Plugin)** 형태로 골라 설치할 수 있습니다. 이는 마치 필요한 공구만 선택해 공구함에 담는 것과 같습니다.
2.  **코루틴 네이티브:** 이것이 Ktor의 심장입니다. Ktor는 태생부터 코루틴을 기반으로 설계되었습니다. 모든 HTTP 요청 핸들러는 코루틴 스코프 내에서 실행되며, 개발자는 API 요청을 처리하는 모든 과정(DB 조회, 외부 API 호출 등)을 `suspend` 함수로 자연스럽게 처리할 수 있습니다. 이는 적은 스레드로 높은 처리량을 감당하는 논블로킹 서버를 매우 쉽게 구현할 수 있음을 의미합니다.
3.  **멀티플랫폼:** Ktor는 서버 프레임워크인 동시에, 15장에서 살펴본 완벽한 멀티플랫폼 **클라이언트**이기도 합니다.

#### 핵심 기능: 플러그인 시스템

Ktor의 거의 모든 기능은 플러그인을 통해 설치됩니다. 필요한 기능만 선택적으로 추가하여 애플리케이션을 가볍게 유지할 수 있습니다.

  * **`Routing`:** HTTP 메서드(GET, POST 등)와 경로(`/hello`, `/users/{id}`)를 정의하는 핵심 플러그인.
  * **`ContentNegotiation`:** `kotlinx.serialization`이나 `Jackson/Gson`과 연동하여 JSON 같은 콘텐츠를 자동으로 객체로 변환하거나 그 반대를 수행.
  * **`Authentication`:** JWT, OAuth, Basic Auth 등 인증 로직을 쉽게 추가.
  * **`CallLogging`:** 모든 수신 요청을 자동으로 로깅.

#### 가장 간단한 Ktor 서버 예제

Ktor 서버가 얼마나 간결한지 확인해 봅시다. 단 하나의 파일로 완전한 비동기 "Hello, World" 서버를 실행할 수 있습니다.

```kotlin
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.response.*
import io.ktor.server.routing.*

fun main() {
    // Netty 엔진을 기반으로 8080 포트에서 임베디드 서버를 시작합니다.
    embeddedServer(Netty, port = 8080) {
        // 라우팅 플러그인 설치 및 설정
        routing {
            // 루트 경로("/")에 대한 GET 요청을 처리
            get("/") {
                // 이 람다 블록은 'suspend' 컨텍스트입니다!
                // 따라서 delay()나 DB 호출 같은 suspend 함수를 직접 호출할 수 있습니다.
                // call.respondText는 비동기적으로 텍스트 응답을 보냅니다.
                call.respondText("Hello, Ktor!")
            }
        }
    }.start(wait = true) // 서버가 메인 스레드를 블로킹하고 계속 실행되도록 함
}
```

`get("/")` 내부의 람다가 `suspend` 컨텍스트라는 점이 핵심입니다. 개발자는 스레드나 콜백을 전혀 신경 쓰지 않고, `delay()`를 호출하든, `withContext(Dispatchers.IO)`로 DB 작업을 하든, 코틀린의 동시성 기능을 마음껏 활용할 수 있습니다. Ktor는 이 모든 것을 논블로킹 방식으로 처리하여 서버의 자원을 극도로 효율적이게 사용합니다.

Ktor는 코틀린의 언어적 특성을 100% 활용하여, 경량의 고성능 마이크로서비스를 빠르고 즐겁게 구축하고자 하는 개발자에게 완벽한 선택입니다.