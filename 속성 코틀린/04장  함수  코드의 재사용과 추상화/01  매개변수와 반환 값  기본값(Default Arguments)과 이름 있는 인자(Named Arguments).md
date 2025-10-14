### 매개변수와 반환 값: 기본값(Default Arguments)과 이름 있는 인자(Named Arguments)

함수의 기본 구조를 익혔으니, 이제 함수의 유연성과 가독성을 극대화하는 코틀린의 강력한 무기, **기본값**과 **이름 있는 인자**에 대해 알아보겠습니다. 이 두 가지 기능은 특히 파라미터가 많은 함수를 다룰 때, 자바의 고질적인 문제였던 생성자 오버로딩의 번거로움을 우아하게 해결해 줍니다.

#### 문제 상황: 수많은 파라미터와 오버로딩의 늪

사용자를 생성하는 함수가 있다고 상상해 봅시다. 필수 정보는 이름과 나이지만, 국적, 언어, 활성 상태 등 선택적인 정보도 받을 수 있습니다.

```java
// Java의 경우
public User(String name, int age) { ... }
public User(String name, int age, String country) { ... }
public User(String name, int age, String country, String language) { ... }
// ... 수많은 오버로딩 함수의 향연
```

이처럼 자바에서는 선택적 파라미터를 처리하기 위해 비슷한 시그니처를 가진 여러 개의 생성자나 메서드를 만드는 \*\*메서드 오버로딩(Method Overloading)\*\*을 사용해야 했습니다. 이는 보일러플레이트 코드를 늘리고 유지보수를 어렵게 만드는 원인이었습니다.

또한 함수를 호출할 때도 문제가 있습니다.
`createUser("Alice", 30, "USA", true)`
이 코드만 보고 `true`가 무엇을 의미하는지 즉시 알 수 있을까요? 아마 함수의 정의를 다시 찾아봐야 할 것입니다.

코틀린은 이 두 가지 문제를 '기본값'과 '이름 있는 인자'로 한 번에 해결합니다.

-----

#### 기본값 (Default Arguments): 파라미터에 기본 옵션 설정하기

코틀린에서는 함수를 정의할 때 파라미터에 **기본값**을 지정할 수 있습니다. 함수를 호출할 때 해당 파라미터에 대한 인자를 전달하지 않으면, 컴파일러가 자동으로 여기에 설정된 기본값을 사용합니다.

```kotlin
fun createUser(
    name: String,
    age: Int,
    country: String = "Korea", // country의 기본값은 "Korea"
    language: String = "Korean", // language의 기본값은 "Korean"
    isActive: Boolean = true // isActive의 기본값은 true
) {
    println("이름: $name, 나이: $age, 국가: $country, 언어: $language, 활성: $isActive")
}
```

이제 이 함수 하나만으로 수많은 오버로딩 함수를 대체할 수 있습니다.

```kotlin
fun main() {
    // 1. 필수 인자만 전달: country, language, isActive는 기본값을 사용
    createUser("Alice", 30)
    // 출력: 이름: Alice, 나이: 30, 국가: Korea, 언어: Korean, 활성: true

    // 2. 일부 선택적 인자만 전달
    createUser("Bob", 25, "USA") // country만 "USA"로 변경, 나머지는 기본값
    // 출력: 이름: Bob, 나이: 25, 국가: USA, 언어: Korean, 활성: true
}
```

더 이상 여러 개의 비슷한 함수를 만들 필요 없이, 단 하나의 함수로 모든 경우의 수를 유연하게 처리할 수 있습니다.

-----

#### 이름 있는 인자 (Named Arguments): 파라미터 이름을 명시하여 인자 전달하기

기본값 기능만으로는 부족한 점이 있습니다. `createUser` 함수에서 `isActive` 값만 `false`로 바꾸고, `country`와 `language`는 기본값을 그대로 쓰고 싶다면 어떻게 해야 할까요? 파라미터의 순서 때문에 원치 않는 `country`와 `language` 값까지 모두 전달해야 할까요?

이때 **이름 있는 인자**가 등장합니다. 이름 있는 인자는 함수를 호출할 때 **`파라미터이름 = 값`** 의 형태로 직접 파라미터를 지정하여 인자를 전달하는 방식입니다.

**장점:**

1.  **순서와 무관:** 파라미터의 순서를 지키지 않아도 됩니다.
2.  **명확성:** 각 값이 어떤 파라미터에 전달되는지 명확히 보여주어 코드의 가독성을 극적으로 향상시킵니다.
3.  **유연성:** 기본값과 함께 사용하면, 원하는 파라미터만 골라서 값을 전달할 수 있습니다.

<!-- end list -->

```kotlin
fun main() {
    // 이름 있는 인자를 사용하여 원하는 파라미터만 골라서 전달
    createUser(name = "Charlie", age = 40, isActive = false)
    // 출력: 이름: Charlie, 나이: 40, 국가: Korea, 언어: Korean, 활성: false

    // 순서를 바꿔서 호출해도 전혀 문제없다!
    createUser(age = 22, isActive = false, name = "David")
    // 출력: 이름: David, 나이: 22, 국가: Korea, 언어: Korean, 활성: false
}
```

일반 인자와 이름 있는 인자를 섞어서 사용할 수도 있습니다. 단, **일반 인자를 먼저 쓰고 그 뒤에 이름 있는 인자를 써야 한다는 규칙**이 있습니다.

```kotlin
// 올바른 사용: 일반 인자(name, age) 뒤에 이름 있는 인자(language)
createUser("Eve", 28, language = "English") 
// 컴파일 오류: 이름 있는 인자 뒤에 일반 인자를 쓸 수 없다.
// createUser(name = "Frank", 33, "German") // Error!
```

기본값과 이름 있는 인자는 단순히 편리한 문법 설탕을 넘어, \*\*빌더 패턴(Builder Pattern)\*\*과 같은 복잡한 디자인 패턴을 언어 차원에서 간결하게 해결해 주는 코틀린의 핵심적인 기능입니다. 이 두 가지를 적극적으로 활용하여 더 읽기 쉽고, 유연하며, 견고한 함수를 설계하는 습관을 들이시기 바랍니다.