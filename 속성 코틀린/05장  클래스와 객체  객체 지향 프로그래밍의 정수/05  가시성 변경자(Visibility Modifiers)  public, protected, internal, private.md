### 가시성 변경자(Visibility Modifiers): public, protected, internal, private

잘 설계된 객체는 자신의 내부는 안전하게 보호하면서, 외부에 공개할 기능만을 선별적으로 노출합니다. 자동차를 운전할 때 우리는 핸들과 페달만 조작할 뿐, 복잡한 엔진 내부를 직접 만지지는 않는 것과 같습니다. 이처럼 객체의 내부 구현을 숨기고 외부에서 접근할 수 있는 범위를 제어하는 것을 \*\*캡슐화(Encapsulation)\*\*라고 하며, 이를 위해 사용되는 키워드가 바로 \*\*가시성 변경자(Visibility Modifier)\*\*입니다.

코틀린은 `public`, `protected`, `internal`, `private` 네 가지 가시성 변경자를 제공합니다.

-----

#### `public`: 어디서든 접근 가능 (기본값)

\*\*`public`\*\*은 가장 개방적인 접근 수준입니다. `public`으로 선언된 프로퍼티나 메서드는 프로젝트 내의 **어디서든** 접근할 수 있습니다.

**코틀린의 중요한 특징:** 클래스나 함수, 프로퍼티 등을 선언할 때 아무런 가시성 변경자를 붙이지 않으면, \*\*기본적으로 `public`\*\*으로 간주됩니다. 이는 `package-private`이 기본값인 자바와의 큰 차이점입니다.

```kotlin
// public은 기본값이므로 보통 생략합니다.
public class Person {
    public var name: String = "홍길동"

    public fun introduce() {
        println("저는 ${name}입니다.")
    }
}
```

-----

#### `private`: 클래스 내부의 비밀

\*\*`private`\*\*는 가장 엄격한 접근 수준입니다. `private`으로 선언된 멤버는 **오직 그 멤버가 선언된 클래스 내부에서만** 접근할 수 있습니다. 자식 클래스에서도 접근이 불가능합니다.

```kotlin
class CoffeeMachine {
    private var waterLevel = 0

    // 외부에서는 이 private 메서드를 직접 호출할 수 없습니다.
    private fun boilWater() {
        println("물을 끓입니다. (현재 수위: ${waterLevel})")
    }

    // public 메서드를 통해 내부 로직을 제어합니다.
    fun brew() {
        if (waterLevel > 0) {
            boilWater()
            println("커피를 내립니다.")
            waterLevel--
        }
    }

    fun addWater(amount: Int) {
        waterLevel += amount
    }
}

fun main() {
    val machine = CoffeeMachine()
    machine.addWater(2)
    machine.brew()
    // machine.waterLevel = 10 // 컴파일 오류! private 멤버에는 외부에서 접근 불가
    // machine.boilWater()    // 컴파일 오류!
}
```

-----

#### `internal`: 모듈 내부의 친구

\*\*`internal`\*\*은 코틀린에만 존재하는 특별하고 매우 유용한 변경자입니다. `internal`로 선언된 멤버는 **같은 모듈(Module) 안에서는 어디서든** 접근할 수 있지만, 다른 모듈에서는 접근할 수 없습니다.

**모듈이란?** 함께 컴파일되는 코틀린 파일들의 묶음을 의미합니다. 예를 들어, 하나의 IntelliJ 모듈, 하나의 Gradle 프로젝트, 하나의 Maven 프로젝트가 각각 하나의 모듈이 됩니다.

`internal`은 라이브러리를 만들 때, 외부에는 공개하고 싶지 않지만 라이브러리 내부의 다른 코드들은 자유롭게 사용해야 하는 '내부 API'를 정의할 때 매우 유용합니다.

```kotlin
// library-core 모듈
package com.mycorp.library

// 이 클래스는 library-core 모듈 안에서만 사용할 수 있습니다.
internal class InternalHelper {
    fun doSomething() { /* ... */ }
}

// 이 클래스는 외부 모듈에서도 사용할 수 있습니다.
public class PublicApi {
    private val helper = InternalHelper() // 같은 모듈이므로 InternalHelper 사용 가능

    fun execute() {
        helper.doSomething()
    }
}
```

`library-core`를 사용하는 다른 `app` 모듈에서는 `PublicApi`는 사용할 수 있지만, `InternalHelper`는 보이지도 않고 사용할 수도 없습니다.

-----

#### `protected`: 자식에게만 물려주는 유산

\*\*`protected`\*\*는 `private`과 비슷하지만, 한 가지 중요한 차이가 있습니다. **선언된 클래스와 그 클래스를 상속받는 자식 클래스 내부에서만** 접근이 가능합니다.

```kotlin
open class Animal {
    protected var health = 100
}

class Dog : Animal() {
    fun heal() {
        // 자식 클래스이므로 부모의 protected 멤버에 접근 가능
        health += 20 
    }
}

fun main() {
    val dog = Dog()
    // dog.health = 50 // 컴파일 오류! 외부에서는 protected 멤버에 접근 불가
}
```

**자바와의 차이점:** 자바의 `protected`는 같은 패키지 내에서도 접근이 가능했지만, 코틀린의 `protected`는 오직 상속 관계에서만 의미를 가지므로 훨씬 더 엄격하고 명확합니다.

-----

#### 한눈에 보기

| 변경자 | 클래스 멤버 | 최상위 선언 |
| :--- | :--- | :--- |
| **`public`** (기본) | 모든 곳 | 모든 곳 |
| **`internal`** | 같은 모듈 내 | 같은 모듈 내 |
| **`protected`** | 클래스, 자식 클래스 | (사용 불가) |
| **`private`** | 같은 클래스 내 | 같은 파일 내 |

> **최상위 선언**: 클래스 밖에 선언된 함수나 변수의 `private`은 해당 파일 내에서만 접근 가능함을 의미합니다.

**캡슐화의 원칙:** 외부에는 최소한의 기능(`public`)만 노출하고, 나머지는 가능한 한 가장 엄격한 접근 수준(`private` \> `internal` \> `protected`)으로 제한하는 것이 좋은 설계의 첫걸음입니다.