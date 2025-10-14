## 구조화된 로깅(Structured Logging): Logback과 JSON 포맷

중앙화된 로깅 시스템을 구축하기로 결정했다면, 다음 단계는 로그를 **'어떤 형식으로'** 남길지 결정하는 것입니다. 단순히 `println`이나 전통적인 텍스트 기반 로그로는 중앙화의 이점을 100% 활용할 수 없습니다.

#### 기존 텍스트 로그의 문제점

일반적인 스프링 부트 애플리케이션의 텍스트 로그는 다음과 같습니다.
`2025-09-24 15:17:00.123 INFO [main] c.e.o.OrderService : Order created with id 123 for member 456`

이 로그는 사람이 읽기에는 좋지만, 기계(Elasticsearch)가 분석하기에는 매우 비효율적입니다.

  * **느린 파싱:** `memberId`가 `456`인 모든 로그를 찾으려면, Elasticsearch는 이 문자열 전체를 대상으로 느리고 깨지기 쉬운 **정규식(Regular Expression)** 파싱을 수행해야 합니다.
  * **타입 정보 부재:** 로그 속 `123`은 숫자가 아닌 그냥 '문자'입니다. "결제 금액이 50000원 이상인 주문 로그" 같은 숫자 기반의 범위 검색이 불가능합니다.
  * **필드 불일치:** A 개발자는 `orderId=123`, B 개발자는 `order_id: 123`과 같이 로그를 남기면, 필드 이름이 달라져 검색과 집계가 불가능해집니다.

-----

### 해답: 구조화된 로깅 (Structured Logging) 📜

**구조화된 로깅**은 로그 메시지를 단순 텍스트가 아닌, \*\*`Key:Value` 쌍으로 이루어진 일관된 형식(주로 JSON)\*\*으로 기록하는 접근 방식입니다.

위의 텍스트 로그를 JSON 형식으로 바꾸면 다음과 같습니다.

```json
{
  "timestamp": "2025-09-24T06:17:00.123Z",
  "level": "INFO",
  "thread_name": "main",
  "logger_name": "com.ecommerce.order.service.OrderService",
  "message": "Order created successfully",
  "context": {
    "orderId": 123,
    "memberId": 456,
    "totalAmount": 52500
  }
}
```

이 방식은 다음과 같은 압도적인 장점을 가집니다.

  * **빠르고 정확한 파싱:** Elasticsearch는 이 JSON을 즉시 파싱하여 각 필드를 인덱싱합니다. 정규식이 필요 없습니다.
  * **강력한 필드 기반 검색:** `context.orderId:123` 또는 `level:ERROR` 와 같이 특정 필드를 대상으로 매우 빠르고 정확한 검색이 가능합니다.
  * **데이터 타입 활용:** `totalAmount`는 '숫자' 타입으로 인덱싱되므로, `context.totalAmount > 50000` 과 같은 범위 검색이나 평균, 합계 같은 집계 연산이 가능해집니다.

-----

### Spring Boot에 JSON 로깅 적용하기: Logback + Logstash Logback Encoder

스프링 부트의 기본 로깅 프레임워크인 `Logback`에 `logstash-logback-encoder` 라이브러리를 추가하여 손쉽게 JSON 로깅을 구현할 수 있습니다.

#### 1\. 의존성 추가 (`build.gradle.kts`)

모든 마이크로서비스(`member-service`, `order-service` 등)에 다음 의존성을 추가합니다.

```kotlin
// build.gradle.kts
dependencies {
    // ...
    implementation("net.logstash.logback:logstash-logback-encoder:7.4")
}
```

#### 2\. Logback 설정 파일 작성 (`logback-spring.xml`)

`src/main/resources` 디렉토리에 `logback-spring.xml` 파일을 생성하여, 로그 출력을 JSON 형식으로 변경하도록 설정합니다.



```xml
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <appender name="CONSOLE_JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>Asia/Seoul</timeZone>
                </timestamp>
                <version/>     <logLevel/>   <message/>    <loggerName/> <threadName/> <mdc/>
                
                <arguments/>

                <stackTrace>
                    <fieldName>stack_trace</fieldName>
                    <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                        <maxDepthPerThrowable>30</maxDepthPerThrowable>
                        <maxLength>2048</maxLength>
                        <rootCauseFirst>true</rootCauseFirst>
                    </throwableConverter>
                </stackTrace>
            </providers>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE_JSON"/>
    </root>
</configuration>
```

#### 3\. 애플리케이션 코드에서 로그 남기기

이제 애플리케이션 코드에서는 평소처럼 `SLF4J` 로거를 사용하되, **Key-Value 쌍을 지원하는 로깅 스타일**을 사용하면 됩니다.

```kotlin
// OrderService.kt
import org.slf4j.LoggerFactory
import org.slf4j.MDC
import net.logstash.logback.argument.StructuredArguments.kv // kv import

// ...

// 1. MDC(Mapped Diagnostic Context)에 요청 전체에 적용될 컨텍스트 정보 추가
// (보통 API 게이트웨이나 인터셉터에서 traceId를 생성하고 MDC에 넣어줍니다)
MDC.put("traceId", "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")

val log = LoggerFactory.getLogger(javaClass)

// 2. kv() 헬퍼를 사용하여 구조화된 인자 전달
log.info("Order process started for member", 
    kv("memberId", 456), 
    kv("productId", 9981)
)

// ...

log.error("Failed to process payment",
    kv("orderId", 123),
    kv("errorCode", "PAYMENT_GATEWAY_TIMEOUT"),
    e // 예외 객체를 마지막 인자로 전달하면 스택 트레이스가 포함됨
)

// 3. 요청 처리가 끝나면 MDC에서 컨텍스트 정보 제거
MDC.clear()
```

  * **MDC (Mapped Diagnostic Context):** `traceId`처럼 하나의 요청 흐름 전체에서 유효한 '컨텍스트 정보'를 담아두는 `ThreadLocal` 기반의 맵입니다. `logback-spring.xml` 설정 덕분에, MDC에 넣어둔 값은 해당 스레드에서 발생하는 **모든 로그에 자동으로 포함**됩니다.

이제 우리의 모든 서비스는 사람이 읽기 편할 뿐만 아니라, 기계가 완벽하게 분석할 수 있는 풍부한 JSON 로그를 `stdout`으로 출력하게 됩니다. 이것으로 중앙화된 로깅 시스템을 구축할 준비가 모두 끝났습니다.