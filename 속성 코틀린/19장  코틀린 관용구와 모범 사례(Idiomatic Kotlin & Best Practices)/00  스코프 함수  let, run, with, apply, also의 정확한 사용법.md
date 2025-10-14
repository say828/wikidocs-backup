### 스코프 함수: let, run, with, apply, also의 정확한 사용법

코틀린 표준 라이브러리에는 `let`, `run`, `with`, `apply`, `also`라는 5개의 스코프 함수가 있습니다. 이 함수들은 코틀린 관용구의 핵심이지만, 동시에 가장 많이 오용되거나 남용되는 기능이기도 합니다.

이 함수들의 공통점은 \*\*'객체의 컨텍스트 내에서 코드 블록(람다)을 실행한다'\*\*는 것입니다. 이들을 정확히 구분하고 사용하기 위해서는 다음 두 가지 기준을 명확히 이해해야 합니다.

1.  **컨텍스트 객체 참조 방식:** 람다 블록 내에서 컨텍스트 객체를 어떻게 참조하는가?
      * **`it` (인자):** `let`, `also` (객체를 람다의 인자로 전달)
      * **`this` (수신 객체):** `run`, `with`, `apply` (객체를 람다의 수신 객체로 사용)
2.  **반환 값:** 스코프 함수 전체의 최종 반환 값은 무엇인가?
      * **람다의 결과:** `let`, `run`, `with` (람다 블록의 마지막 표현식이 반환됨)
      * **컨텍스트 객체:** `apply`, `also` (람다의 결과와 상관없이, 컨텍스트 객체 '자신'이 반환됨)

이 두 가지 기준을 조합하면 다음과 같은 명확한 결정 매트릭스가 완성됩니다.

| 함수 | 컨텍스트 객체 | 반환 값 | 핵심 용도 |
| :--- | :---: | :---: | :--- |
| **`let`** | `it` | 람다 결과 | **널 검사** 후 다른 타입으로 변환 (`?.let`) |
| **`run`** | `this` | 람다 결과 | 객체에 대한 연산 수행 후 특정 값 반환 |
| **`with`** | `this` | 람다 결과 | `run`과 동일 (확장 함수가 아님) |
| **`apply`** | `this` | **컨텍스트 객체** | **객체 생성 및 초기 설정** (빌더 패턴) |
| **`also`** | `it` | **컨텍스트 객체** | **부가적인 작업** (로깅, 데이터 추가 등 사이드 이펙트) |

-----

#### 1\. `let` (it -\> Lambda Result)

`let`의 존재 이유는 99% \*\*널 안전 호출(`?.`)\*\*과 결합하기 위함입니다. 널이 아닐 경우에만 코드 블록을 실행하고, 해당 객체를 `it`으로 전달받아 사용합니다.

```kotlin
val name: String? = getUserName()

name?.let { nonNullName -> // 'name'이 null이 아닐 때만 실행
    // 이 블록 안에서 'it'(또는 nonNullName)은 Non-null String 타입입니다.
    println("이름: $nonNullName")
}
```

#### 2\. `apply` (this -\> Context Object)

`apply`는 "객체를 생성하고, 이 객체를 대상으로 즉시 초기 설정을 적용한 뒤, 바로 그 객체 자신을 반환"할 때 사용합니다. 객체 초기화를 위한 빌더 패턴을 완벽하게 대체합니다.

```kotlin
// Intent 객체를 생성함과 '동시에' 설정을 적용(apply)함
val intent = Intent(this, SecondActivity::class.java).apply {
    // 이 블록의 'this'는 Intent 객체입니다.
    putExtra("EXTRA_USER_ID", 123)
    putExtra("EXTRA_MODE", "EDIT")
    action = "START_EDITING"
}
// intent 변수에는 모든 설정이 완료된 Intent 객체가 담겨 있습니다.
startActivity(intent)
```

#### 3\. `run` (this -\> Lambda Result)

`run`은 `apply`와 비슷하게 객체를 `this`로 받지만, 객체 자신이 아닌 **람다의 마지막 실행 결과를 반환**합니다. 특정 객체가 필요한 연산을 수행하고 그 '결과'를 변수에 담을 때 유용합니다.

```kotlin
val person = Person("Alice", 30)
val description: String = person.run {
    // 'this'는 person 객체
    calculateAgeInMonths() // 객체의 메서드 호출
    "이름: $name, 나이: $age" // 이 String 값이 description에 할당됨
}
```

#### 4\. `also` (it -\> Context Object)

`also`는 "객체를 받아, 그 객체로 무언가 부가적인(side-effect) 작업을 수행한 뒤, 다시 그 객체 자신을 반환"할 때 씁니다. '그리고 또한(also)'이라는 이름처럼, 메인 로직의 흐름을 방해하지 않고 중간에 끼어드는 작업(주로 로깅)에 완벽합니다.

```kotlin
val user = userRepository.createUser("Alice").also { newUser ->
    // 'it'(또는 newUser)은 방금 생성된 User 객체입니다.
    log.info("새로운 사용자가 생성되었습니다: ID = ${newUser.id}")
    eventBus.post(UserCreatedEvent(newUser))
}
// 'user' 변수에는 로깅과 이벤트 발행 작업이 끝난 원본 User 객체가 담깁니다.
```

#### 5\. `with` (this -\> Lambda Result)

`with`는 `run`과 기능이 완전히 동일하지만, 확장 함수가 아니라 일반 함수라는 점만 다릅니다. 즉, `object.run { ... }` 대신 `with(object) { ... }` 형태로 호출합니다. 널이 아닌 객체에 대해 여러 작업을 수행해야 할 때 코드를 묶어주는 역할을 합니다.

```kotlin
val person = Person("Bob", 40)
val summary = with(person) {
    // 'this'는 person 객체
    "요약: $name ($age세)"
}
```

**황금률:** 스코프 함수를 사용하는 것이 일반 변수 선언보다 코드를 더 복잡하고 읽기 어렵게 만든다면, 절대 사용하지 마십시오. 이 함수들은 코드를 더 유창하게 만들기 위한 도구이지, 복잡하게 만들기 위한 도구가 아닙니다.