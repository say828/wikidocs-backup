## 코루틴(Coroutines) 입문: 스레드 모델의 한계와 비동기/논블로킹의 필요성

지금까지 우리가 살펴본 코틀린의 기능들(data class, null safety, extensions, collections)은 '코드의 가독성'과 '안정성'을 높여주었습니다. 이제 마지막으로 '성능', 특히 **대규모 동시성(Concurrency)** 문제에 대한 코틀린의 해답을 살펴볼 차례입니다.

-----

### 전통적인 '1 스레드 - 1 요청' 모델의 한계

우리가 일반적으로 사용하는 스프링 부트(Spring Boot MVC)는 내장 톰캣(Tomcat)을 사용하며, 이는 **'1 요청(Request) 당 1 스레드(Thread)'** 모델을 기반으로 동작합니다.

이커머스 MSA 환경에서 이 모델이 어떻게 동작하는지 '주문' 시나리오로 살펴보겠습니다.

1.  사용자가 '주문 생성' API를 호출합니다. (`Request`)
2.  톰캣은 스레드 풀(Thread Pool)에서 \*\*`스레드 A`\*\*를 할당하여 `OrderService`의 로직을 실행합니다.
3.  `OrderService`는 주문을 생성하기 위해 `ProductService`에 '상품 재고 확인' API를 호출합니다. (네트워크 I/O 발생)
4.  \*\*`스레드 A`\*\*는 `ProductService`의 응답이 올 때까지 **'대기(Blocking)'** 상태에 빠집니다. 이 스레드는 응답이 오기 전까지 CPU를 사용하지도 않으면서, 메모리만 차지한 채 아무 일도 하지 않습니다.
5.  `ProductService`가 200ms 뒤에 응답합니다.
6.  `스레드 A`가 '대기' 상태에서 깨어나 나머지 로직을 처리하고 응답을 반환합니다.

평상시에는 이 모델이 문제없습니다. 하지만 '블랙 프라이데이' 세일로 **동시 접속자 10,000명**이 몰리면 어떻게 될까요?

  * 톰캣의 기본 스레드 풀 크기(예: 200개)는 순식간에 고갈됩니다.
  * 200개의 스레드가 모두 `ProductService`나 `PaymentService`를 호출하며 '대기(Blocking)' 상태에 빠집니다.
  * 나머지 9,800개의 요청은 스레드를 할당받지 못해 대기 큐(Queue)에서 하염없이 기다리거나 'Connection Timeout' 에러를 반환합니다.
  * 시스템은 사실상 마비(Crash)됩니다.

이것이 바로 **스레드 고갈(Thread Exhaustion)** 문제이며, 전통적인 스레드 모델이 대규모 트래픽을 처리하지 못하는 이유입니다.

-----

### 비동기/논블로킹(Async/Non-Blocking)의 필요성

이 문제를 해결하려면 `스레드 A`가 '대기'하지 않고 다른 일을 하러 가야 합니다.

1.  `스레드 A`가 `ProductService`에 '상품 재고 확인'을 \*\*요청(Request)\*\*합니다.
2.  `스레드 A`는 응답을 기다리지 않고 **즉시 스레드 풀로 반납**됩니다. 🚀
3.  반납된 `스레드 A`는 \*\*다른 사용자(User B)\*\*의 요청을 받아 처리하기 시작합니다.
4.  200ms 뒤 `ProductService`의 응답이 도착하면, 스레드 풀의 \*\*다른 유휴 스레드(예: `스레드 B`)\*\*가 이 응답을 받아 원래 요청의 나머지 로직을 처리합니다.

이 방식에서는 CPU 코어 개수(예: 8개)와 비슷한 매우 적은 수의 스레드만으로도 수만 개의 동시 요청을 효율적으로 처리할 수 있습니다. 스레드가 '노는(Blocking)' 시간이 없기 때문입니다.

### 코루틴: 동기식 코드의 가독성 + 비동기식 코드의 성능

이러한 논블로킹 아키텍처를 구현하기 위해 자바 진영에서는 `CompletableFuture` (콜백 헬)나 Project Reactor (`Mono`, `Flux`) 같은 리액티브 프로그래밍을 사용했습니다. 하지만 이 방식들은 코드가 복잡해지고 학습 곡선이 매우 가파르다는 치명적인 단점이 있습니다.

\*\*코틀린 코루틴(Coroutines)\*\*은 이 문제의 완벽한 대안입니다.

코루틴은 OS가 관리하는 비싼 '스레드'가 아닌, 코틀린 런타임이 관리하는 매우 가벼운 '작업 단위'(Lightweight Thread)입니다. 수백만 개를 만들어도 부담이 없습니다.

코루틴의 핵심은 `suspend` (일시 중단) 키워드입니다.

#### 1\. 전통적인 '블로킹(Blocking)' 코드

```kotlin
// 이 코드는 '스레드'를 멈춥니다.
fun getOrderDetails(orderId: Long): OrderResponse {
    // 1. 여기서 100ms 스레드 대기
    val order = orderRepository.findById(orderId) 
    // 2. 여기서 200ms 스레드 대기
    val member = memberClient.getMember(order.memberId) 
    
    return OrderResponse(order, member)
}
```

#### 2\. 코루틴 기반 '논블로킹(Non-Blocking)' 코드

```kotlin
// 이 코드는 '스레드'를 멈추지 않고, '코루틴'만 일시 중단합니다.
// (12장에서 자세히 다룰 WebFlux + Coroutines 예시)
suspend fun getOrderDetails(orderId: Long): OrderResponse {
    // 1. 여기서 '코루틴'이 일시 중단됨. (스레드는 반납됨)
    val order = orderRepository.findById(orderId) // suspend 함수
    
    // 2. 응답이 오면 다른 스레드가 여기서부터 이어서 실행
    // 3. 여기서 다시 '코루틴' 일시 중단. (스레드는 반납됨)
    val member = memberClient.getMember(order.memberId) // suspend 함수
    
    // 4. 응답이 오면 다른 스레드가 이어서 실행
    return OrderResponse(order, member)
}
```

놀랍게도 **두 코드의 로직과 형태는 거의 동일합니다.**

코루틴을 사용하면, 개발자는 마치 동기식/블로킹 코드를 짜듯이 **순차적으로** 코드를 작성할 수 있습니다. 하지만 `suspend` 키워드가 붙은 I/O 지점에서, 코틀린 컴파일러가 알아서 코루틴을 중단시키고 스레드를 반납하는 '논블로킹' 코드로 변환해 줍니다.

이는 **비동기 프로그래밍의 성능**과 **동기 프로그래밍의 가독성**이라는 두 마리 토끼를 모두 잡는 혁신입니다.

이것으로 코틀린의 핵심 기능을 마칩니다. 우리는 12장에서 이 코루틴을 스프링 WebFlux와 R2DBC에 적용하여 이커머스 MSA 전체를 고성능 논블로킹 시스템으로 전환할 것입니다.