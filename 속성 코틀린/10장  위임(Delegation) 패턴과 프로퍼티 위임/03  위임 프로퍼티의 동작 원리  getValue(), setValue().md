### 위임 프로퍼티의 동작 원리: getValue(), setValue()

`by` 키워드를 사용한 프로퍼티 위임이 마법처럼 느껴질 수 있지만, 그 내부에는 코틀린 컴파일러가 따라가는 명확하고 일관된 규칙이 숨어있습니다. 이 규칙의 핵심이 바로 `getValue()`와 `setValue()`라는 두 약속된 함수입니다.

`val p by delegate` 와 같은 코드를 볼 때, 컴파일러는 프로퍼티 `p`에 대한 모든 접근을 `delegate` 객체의 `getValue()`와 `setValue()` 함수 호출로 변환합니다.

-----

#### `val` 프로퍼티 (읽기 전용)의 동작

읽기 전용 프로퍼티 `val`을 위임할 경우, 컴파일러는 프로퍼티의 값을 **읽는** 시점에 대리인(delegate)의 `getValue()` 함수를 호출합니다.

```kotlin
class Resource {
    // p라는 String 타입 프로퍼티의 GET 접근을 resourceDelegate에게 위임
    val p: String by ResourceDelegate()
}

class ResourceDelegate {
    // 약속된 시그니처의 getValue 함수
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, '${property.name}' 프로퍼티의 값을 읽습니다."
    }
}
```

이제 `Resource`의 인스턴스를 만들고 `p` 프로퍼티에 접근하면,

```kotlin
val res = Resource()
println(res.p) // res.p 값을 읽는 코드
```

컴파일러는 `res.p` 코드를 내부적으로 다음과 같이 변환합니다.

`resourceDelegate.getValue(res, ::p)`

  * **`resourceDelegate`**: `by` 키워드 뒤에 있는 대리인 객체입니다.
  * **`getValue()`**: 대리인이 반드시 제공해야 하는 약속된 함수입니다.
  * **`res`**: 첫 번째 인자로, 프로퍼티를 소유한 객체(`Resource`의 인스턴스) 자신(`thisRef`)이 전달됩니다.
  * **`::p`**: 두 번째 인자로, 프로퍼티 자체에 대한 정보(이름, 타입 등)를 담고 있는 `KProperty` 타입의 객체가 전달됩니다.

따라서 위 `println(res.p)`의 실행 결과는 다음과 같습니다.

```
Resource@..., 'p' 프로퍼티의 값을 읽습니다.
```

-----

#### `var` 프로퍼티 (변경 가능)의 동작

변경 가능 프로퍼티 `var`를 위임할 경우, 값을 읽을 때는 `getValue()`가, 값을 쓸 때는 `setValue()`가 호출됩니다.

```kotlin
class User {
    // name 프로퍼티의 GET/SET 접근을 NameDelegate에게 위임
    var name: String by NameDelegate()
}

class NameDelegate {
    private var value: String = "Default"
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        println("${property.name} 값을 읽습니다.")
        return value
    }
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: String) {
        println("${property.name} 값을 '${newValue}'(으)로 설정합니다.")
        value = newValue
    }
}
```

이제 `User` 객체의 `name` 프로퍼티를 사용하면,

```kotlin
val user = User()

// 1. 쓰기(Set)
user.name = "Alice" // "name 값을 'Alice'(으)로 설정합니다." 출력

// 2. 읽기(Get)
println(user.name)  // "name 값을 읽습니다." 출력 후 "Alice" 출력
```

컴파일러는 각 코드를 다음과 같이 변환합니다.

1.  **`user.name = "Alice"`** ➡️  **`nameDelegate.setValue(user, ::name, "Alice")`**
2.  **`println(user.name)`** ➡️ **`println(nameDelegate.getValue(user, ::name))`**

이처럼 프로퍼티 위임은 컴파일러가 정해진 규칙에 따라 `getValue`와 `setValue` 함수 호출 코드를 자동으로 생성해주는 편리한 기능입니다. 대리인 클래스는 특정 인터페이스를 구현할 필요 없이, 이 약속된 시그니처의 함수들만 `operator` 키워드와 함께 제공하면 됩니다. 이 덕분에 우리는 매우 유연하고 가벼운 커스텀 대리인을 손쉽게 만들 수 있습니다.

-----

이것으로 코드 재사용의 패러다임을 확장하는 위임 패턴에 대한 10장을 마칩니다. 클래스 위임은 상속의 강력한 대안을, 프로퍼티 위임은 프로퍼티 관련 로직을 재사용하는 우아한 방법을 제시합니다. 이제 당신은 코틀린이 제공하는 정교한 도구들을 사용하여 더 유연하고 유지보수하기 좋은 설계를 할 수 있는 능력을 갖추게 되었습니다.