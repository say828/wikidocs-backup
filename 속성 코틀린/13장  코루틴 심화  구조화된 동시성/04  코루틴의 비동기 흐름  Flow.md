### 코루틴의 비동기 흐름: Flow

지금까지 우리가 다룬 `suspend` 함수나 `async` 블록은 비동기적으로 **단일 값**을 반환하는 데 탁월했습니다. 네트워크에서 사용자 정보 '하나'를 가져오거나, 복잡한 연산 결과 '하나'를 계산하는 작업들이었죠.

하지만 만약 비동기적으로 **여러 개의 값**을 순차적으로 받아와야 한다면 어떻게 해야 할까요? 예를 들어, 실시간으로 변하는 주식 가격을 계속 수신하거나, 대용량 파일을 조각내어 순서대로 읽거나, 사용자의 위치 정보를 지속적으로 업데이트받는 상황입니다.

이러한 \*\*비동기 데이터 스트림(Asynchronous Data Stream)\*\*을 다루기 위해 코틀린 코루틴이 제공하는 해답이 바로 \*\*`Flow`\*\*입니다. `Flow`는 RxJava와 같은 반응형 프로그래밍 라이브러리의 개념을 코루틴의 세계에 가장 잘 녹여낸 도구입니다.

-----

#### Flow란 무엇인가?

**Flow**는 정해지지 않은 시간 동안 \*\*여러 개의 값을 순차적으로 생성(emit)\*\*할 수 있는 타입입니다.

**비유:**

  * **`suspend` 함수**는 '단품 요리 주문' 🍔과 같습니다. 주문하고 기다리면, 잠시 후 요리 **하나**가 나옵니다.
  * \*\*`Flow`\*\*는 '컨베이어 벨트 위의 초밥' 🍣과 같습니다. 벨트가 돌아가는 동안, 초밥 **여러 개**가 시간 간격을 두고 내 앞에 하나씩 도착합니다.

##### Flow의 핵심 특징: Cold Stream

`Flow`는 \*\*콜드 스트림(Cold Stream)\*\*입니다. 이는 매우 중요한 특징으로, `Flow`를 생성하는 코드 블록은 누군가가 값을 요청하기 전까지는 **절대 실행되지 않는다**는 의미입니다. 마치 요리 레시피(Flow)는 그 자체로는 아무 일도 하지 않고, 누군가 그 레시피를 보고 요리를 시작(`collect`)해야만 비로소 재료를 손질하기 시작하는 것과 같습니다.

-----

#### Flow 생성하고 사용하기

##### 1\. Flow 생성: `flow` 빌더

가장 기본적인 Flow 생성 방법은 `flow { ... }` 빌더를 사용하는 것입니다.

  * 빌더 내부의 람다는 `suspend` 컨텍스트이므로, `delay`와 같은 일시 중단 함수를 호출할 수 있습니다.
  * **`emit()`** 함수를 사용하여 스트림에 새로운 값을 방출(생산)합니다.

<!-- end list -->

```kotlin
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

// 1초마다 1부터 3까지의 숫자를 방출하는 Flow를 생성하는 함수
fun simpleFlow(): Flow<Int> = flow {
    println("Flow가 시작되었습니다.")
    for (i in 1..3) {
        delay(1000L) // 1초 대기
        emit(i)      // 값을 방출
    }
}
```

`simpleFlow()` 함수를 호출하는 것만으로는 `println`이나 `delay`, `emit` 어떤 코드도 실행되지 않습니다. 아직 아무도 '요리 시작\!'을 외치지 않았기 때문입니다.

##### 2\. Flow 소비: `collect` 연산자

Flow가 값을 방출하도록 하려면, \*\*종단 연산자(Terminal Operator)\*\*를 사용하여 Flow를 '소비' 또는 '수집'해야 합니다. 가장 기본적인 종단 연산자가 바로 \*\*`collect`\*\*입니다.

`collect`는 일시 중단 함수이며, Flow가 모든 값을 방출하고 **완료될 때까지** 현재 코루틴을 일시 중단시킵니다.

```kotlin
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.flow.collect

fun main() = runBlocking {
    println("메인: collect를 호출합니다...")
    val flow = simpleFlow()
    
    // collect를 호출하는 순간, 비로소 flow { ... } 블록의 코드가 실행되기 시작합니다.
    flow.collect { value ->
        println("메인: 수집된 값 -> $value")
    }
    
    println("메인: collect가 완료되었습니다.")
}
```

**실행 결과:**

```
메인: collect를 호출합니다...
Flow가 시작되었습니다.
(1초 후)
메인: 수집된 값 -> 1
(1초 후)
메인: 수집된 값 -> 2
(1초 후)
메인: 수집된 값 -> 3
메인: collect가 완료되었습니다.
```

"Flow가 시작되었습니다." 메시지가 `collect` 호출 시점에 출력되는 것을 통해 `Flow`가 콜드 스트림임을 명확히 알 수 있습니다.

-----

#### 중간 연산자: 스트림 가공하기

`Flow`의 진정한 강력함은 스트림을 가공하는 \*\*중간 연산자(Intermediate Operators)\*\*를 체인 형태로 연결할 때 나타납니다. `map`, `filter`와 같은 중간 연산자들은 스트림의 각 값을 변환하거나 필터링하는 새로운 `Flow`를 반환합니다. 이들 역시 콜드(cold) 속성을 가지며, 최종적으로 종단 연산자가 호출될 때 순차적으로 적용됩니다.

```kotlin
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    (1..5).asFlow() // 1부터 5까지의 숫자를 방출하는 Flow 생성
        .filter { // 중간 연산자 1: 필터링
            println("Filter: $it")
            it % 2 != 0 // 홀수만 통과시킨다
        }
        .map { // 중간 연산자 2: 변환
            println("Map: $it")
            "숫자 [${it}]" // 숫자를 문자열로 변환한다
        }
        .collect { result -> // 종단 연산자: 최종 결과를 소비
            println("Collect: $result")
        }
}
```

`Flow`는 비동기적으로 발생하는 여러 개의 데이터를 동기적인 코드처럼 다룰 수 있게 해주는 혁신적인 도구입니다. 복잡한 콜백 없이도, 풍부한 연산자들을 통해 비동기 데이터 스트림을 선언적이고 직관적으로 처리할 수 있습니다.