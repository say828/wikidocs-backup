## 코틀린과 AWS Lambda/Azure Functions: 클라우드 FaaS 구현

이론을 넘어, 이제 실제 코드를 통해 코틀린으로 서버리스 함수를 작성하고, 대표적인 FaaS(Function-as-a-Service) 플랫폼인 **AWS Lambda**에 배포하는 과정을 알아보겠습니다. Azure Functions와 Google Cloud Functions 또한 유사한 모델을 따릅니다.

-----

### AWS Lambda의 프로그래밍 모델

AWS Lambda에서 JVM 기반(코틀린, 자바) 함수를 작성하려면, AWS가 제공하는 표준 **핸들러(Handler) 인터페이스**를 구현해야 합니다. 이 인터페이스는 Lambda 런타임이 우리 코드를 호출하기 위한 '진입점(Entry Point)' 역할을 합니다.

가장 일반적으로 사용되는 인터페이스는 `RequestHandler<I, O>` 입니다.

  * `handleRequest(I input, Context context): O`
      * `I (Input)`: 함수를 트리거한 이벤트의 데이터 타입입니다. (예: SQS 메시지, API Gateway의 HTTP 요청)
      * `O (Output)`: 함수의 반환 타입입니다.
      * `Context`: 함수 실행에 대한 런타임 메타데이터(남은 실행 시간, 요청 ID 등)를 담고 있는 객체입니다.

-----

### 실전 예제: '주문 완료 이메일 발송' 함수 구현 (AWS Lambda)

**시나리오:** Kafka의 `ecommerce.orders.paid` 토픽에 `OrderPaidEvent`가 발행되면, 이를 트리거로 AWS Lambda 함수를 실행하여 고객에게 이메일을 발송합니다.
(참고: Kafka 토픽과 Lambda를 직접 연결하거나, 보통 중간에 AWS SQS 같은 큐를 두어 더 안정적인 파이프라인을 구성합니다. 여기서는 SQS 큐가 트리거라고 가정합니다.)

#### 1\. 의존성 추가 (`build.gradle.kts`)

서버리스 함수를 위한 새로운 Gradle 프로젝트(`notification-lambda`)를 생성합니다.

```kotlin
// notification-lambda/build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.24"
    // 1. 모든 의존성을 포함하는 'Fat Jar'를 만들기 위한 Shadow Jar 플러그인
    id("com.github.johnrengelman.shadow") version "8.1.1"
}

dependencies {
    // 2. AWS Lambda의 핵심 핸들러 인터페이스 포함
    implementation("com.amazonaws:aws-lambda-java-core:1.2.3")
    // 3. SQS, S3 등 표준 이벤트 객체 포함
    implementation("com.amazonaws:aws-lambda-java-events:3.11.5")
    
    // JSON 직렬화/역직렬화를 위한 Jackson
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.15.2")
}
```

#### 2\. 핸들러 함수 작성 (`NotificationHandler.kt`)

`RequestHandler` 인터페이스를 구현하여, SQS 이벤트가 도착했을 때 이메일을 발송하는 로직을 작성합니다.

```kotlin
// notification-lambda/src/main/kotlin/com/ecommerce/notification/NotificationHandler.kt
package com.ecommerce.notification

import com.amazonaws.services.lambda.runtime.Context
import com.amazonaws.services.lambda.runtime.RequestHandler
import com.amazonaws.services.lambda.runtime.events.SQSEvent
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper
import com.fasterxml.jackson.module.kotlin.readValue

// 1. SQS 이벤트를 입력(I)으로 받고, 성공/실패 문자열을 출력(O)으로 반환
class NotificationHandler : RequestHandler<SQSEvent, String> {
    
    // (실제로는 DI 프레임워크를 사용하거나 직접 생성)
    private val emailSender = EmailSender() 
    private val objectMapper = jacksonObjectMapper()

    override fun handleRequest(input: SQSEvent, context: Context): String {
        val logger = context.logger // Lambda 런타임이 제공하는 로거
        logger.log("Received ${input.records.size} SQS messages.")

        // 2. SQS는 여러 메시지를 배치로 전달할 수 있음
        input.records.forEach { message ->
            try {
                logger.log("Processing message: ${message.messageId}")
                
                // 3. SQS 메시지 본문(JSON 문자열)을 우리 이벤트 객체로 역직렬화
                val event = objectMapper.readValue<OrderPaidEvent>(message.body)
                
                // 4. 비즈니스 로직 호출
                emailSender.sendOrderConfirmationEmail(event)

            } catch (e: Exception) {
                // 실패 시 로그를 남기고 다음 메시지 처리 (에러 핸들링 정책에 따라 다름)
                logger.log("ERROR processing message ${message.messageId}: ${e.message}")
            }
        }
        return "Successfully processed ${input.records.size} messages."
    }
}

// 이메일 발송 로직을 담당하는 가상 클래스
class EmailSender {
    fun sendOrderConfirmationEmail(event: OrderPaidEvent) {
        println("Sending email to member ${event.memberId} for order ${event.orderId}...")
        // ... (외부 이메일 서비스 API 호출 로직) ...
        Thread.sleep(100) // 네트워크 호출 흉내
        println("Email sent successfully!")
    }
}

// SQS 메시지 Body에 담겨 올 이벤트 객체
data class OrderPaidEvent(val orderId: Long, val memberId: Long, val totalAmount: Long)
```

#### 3\. 패키징 및 배포

1.  **패키징:** Gradle Shadow 플러그인을 사용하여 모든 의존성이 포함된 단일 `.jar` 파일을 생성합니다.

    ```bash
    ./gradlew shadowJar
    ```

    `build/libs` 디렉토리에 `notification-lambda-1.0-all.jar`와 같은 파일이 생성됩니다.

2.  **배포 (AWS Console 기준):**

      * AWS Lambda 콘솔에서 '함수 생성'을 선택합니다.
      * 런타임으로 'Java 17' (또는 코틀린 버전에 맞는)을 선택합니다.
      * 위에서 생성한 'fat jar' 파일을 업로드합니다.
      * **'핸들러' 설정**에 우리 코드의 진입점 경로를 입력합니다: `com.ecommerce.notification.NotificationHandler::handleRequest`
      * '트리거 추가'를 선택하고, `OrderPaidEvent`가 담겨 올 SQS 큐를 지정합니다.
      * 필요한 환경 변수(이메일 서비스 API 키 등)를 설정하고 함수를 배포합니다.

이제 Kafka로부터 `OrderPaidEvent`가 SQS를 통해 전달될 때마다, AWS Lambda는 이 함수를 자동으로 실행하여 이메일을 발송할 것입니다.

-----

### Azure Functions와 Google Cloud Functions

다른 주요 클라우드 FaaS 플랫폼들도 매우 유사한 프로그래밍 모델을 제공합니다.

  * **Azure Functions:** `@FunctionName`, `@QueueTrigger`와 같은 어노테이션 기반으로 함수를 정의하여, 코드만 보면 어떤 이벤트에 의해 트리거되는지 더 명확하게 알 수 있는 장점이 있습니다.
  * **Google Cloud Functions:** `HttpFunction`이나 `BackgroundFunction` 인터페이스를 구현하여, HTTP 트리거 또는 Pub/Sub 같은 백그라운드 이벤트 트리거를 처리합니다.

플랫폼별 API는 조금씩 다르지만, \*\*"이벤트에 의해 트리거되는 짧은 생명주기의 상태 없는 함수"\*\*라는 핵심 패러다임은 모두 동일합니다.