## 코루틴(Coroutines)이 Reactive보다 나은 이유: 절차적 프로그래밍의 가독성

Spring WebFlux와 프로젝트 리액터(`Mono`, `Flux`)는 의심할 여지 없이 강력한 고성능 논블로킹 솔루션입니다. 하지만 12장 01절의 마지막에서 보았듯, `flatMap`과 `map` 연산자가 끝없이 체이닝되는 함수형 코드 스타일은 개발자에게 새로운 인지적 부담을 안겨줍니다. 코드는 비즈니스 로직을 서술하기보다, 데이터 파이프라인의 '조립 설명서'처럼 변해버립니다.

> **"논블로킹의 성능은 얻고 싶은데, 예전의 간단하고 읽기 쉬운 코드로 돌아갈 수는 없을까?"**

이 질문에 대한 코틀린의 대답이 바로 \*\*코루틴(Coroutines)\*\*입니다.

-----

### 코루틴: 컴파일러가 마법을 부리는 비동기 코드

코루틴은 OS가 관리하는 무거운 '스레드'가 아닌, 코틀린 런타임이 관리하는 \*\*깃털처럼 가벼운 '작업 단위'(Light-weight thread)\*\*입니다. 수백만 개를 만들어도 시스템에 거의 부담을 주지 않습니다.

코루틴의 진정한 마법은 \*\*`suspend` (일시 중단)\*\*라는 키워드 하나에 있습니다. 개발자가 함수 앞에 `suspend`를 붙이면, 코틀린 컴파일러는 이 함수를 특별한 방식으로 컴파일합니다. 함수 내부에서 다른 `suspend` 함수(네트워크 I/O, DB I/O 등)를 만나면, 스레드를 \*\*'블로킹'\*\*시키는 대신, **'코루틴의 실행을 잠시 멈추고(suspend)'** 스레드를 즉시 다른 작업을 위해 반납하도록 코드를 변환합니다. 나중에 I/O 작업이 완료되면, 비어있는 스레드 중 하나를 잡아 중단되었던 지점부터 작업을 재개합니다.

이 모든 복잡한 과정은 **컴파일러**가 알아서 처리해 줍니다. 개발자는 그저 비동기 작업이 필요한 함수에 `suspend` 키워드를 붙여주기만 하면 됩니다.

-----

### 코드 비교: 블로킹 vs 리액티브 vs 코루틴

**시나리오:** ID로 주문(`Order`)을 조회한 뒤, 그 주문을 한 회원(`Member`)의 정보를 다시 조회하여 최종 DTO로 조합하기

#### 1\. 전통적인 Spring MVC (블로킹) - **가장 읽기 쉽다**

```kotlin
fun getOrderWithMember(orderId: Long): OrderWithMemberDto {
    // 1. 주문을 블로킹 방식으로 조회 (여기서 스레드가 멈춤)
    val order = orderRepository.findById(orderId)
    
    // 2. 회원 정보를 블로킹 방식으로 조회 (여기서 스레드가 또 멈춤)
    val member = memberRepository.findById(order.memberId)

    // 3. 조합하여 반환
    return OrderWithMemberDto(order, member)
}
```

#### 2\. Spring WebFlux (리액티브) - **가장 복잡하다**

```kotlin
fun getOrderWithMember(orderId: Long): Mono<OrderWithMemberDto> {
    return orderRepository.findById(orderId) // Mono<Order>
        .flatMap { order -> // flatMap으로 비동기 작업을 연결
            memberRepository.findById(order.memberId) // Mono<Member>
                .map { member -> OrderWithMemberDto(order, member) } // map으로 조합
        }
}
```

#### 3\. Spring + Coroutines (코루틴) - **블로킹 코드와 거의 똑같다\!**

```kotlin
// suspend 함수는 다른 suspend 함수나 코루틴 스코프 안에서만 호출 가능
suspend fun getOrderWithMember(orderId: Long): OrderWithMemberDto {
    // 1. 주문을 논블로킹 방식으로 조회 (여기서 코루틴이 '일시 중단'됨)
    val order = orderRepository.findById(orderId) // suspend 함수
    
    // 2. 회원 정보를 논블로킹 방식으로 조회 (여기서 코루틴이 '일시 중단'됨)
    val member = memberRepository.findById(order.memberId) // suspend 함수

    // 3. 조합하여 반환
    return OrderWithMemberDto(order, member)
}
```

**결과는 놀랍습니다.** 코루틴 버전의 코드는 `suspend` 키워드가 추가된 것을 제외하면, 우리가 가장 이해하기 쉬운 **1번 블로킹 코드와 문법적으로 100% 동일**합니다.

-----

### 코루틴이 Reactive보다 나은 이유

1.  **압도적인 가독성과 단순성 (Readability & Simplicity):**
    `flatMap`이나 `map` 같은 복잡한 연산자 체인 없이, 마치 동기식 코드를 짜듯이 **절차적(Imperative)이고 순차적인 프로그래밍**이 가능합니다. 비즈니스 로직을 그대로 코드로 옮길 수 있어 가독성이 극대화되고 유지보수가 쉬워집니다.

2.  **쉬운 디버깅 (Easy Debugging):**
    리액티브 코드에서 예외가 발생하면, 수많은 연산자를 거친 복잡한 스택 트레이스(Stack Trace)가 출력되어 원인을 찾기 매우 어렵습니다. 반면, 코루틴은 코드의 순차적인 컨텍스트를 유지하므로 **스택 트레이스가 블로킹 코드와 거의 유사하게** 나타나 디버깅이 훨씬 직관적입니다.

3.  **구조화된 동시성 (Structured Concurrency):**
    코루틴은 `CoroutineScope`라는 개념을 통해 여러 비동기 작업의 생명주기(Lifecycle)를 부모-자식 관계로 묶어서 관리할 수 있습니다. 이를 통해 "중간에 에러가 나면 관련된 모든 작업을 한 번에 취소"하거나, "모든 작업이 끝날 때까지 기다리는" 등의 복잡한 동시성 제어를 매우 쉽게 구현할 수 있습니다.

**결론적으로, 코루틴은 리액티브 스트림즈가 제공하는 '논블로킹의 성능'과 전통적인 블로킹 코드가 제공하는 '절차적 프로그래밍의 가독성'이라는 두 마리 토끼를 모두 잡은, 코틀린의 진정한 '킬러 기능'입니다.**

다음 절부터 우리는 이 코루틴을 실제 Spring Boot WebFlux 환경에서 어떻게 사용하고, R2DBC를 통해 DB 접근까지 논블로킹으로 처리하는지 실전 코드를 통해 배워보겠습니다.