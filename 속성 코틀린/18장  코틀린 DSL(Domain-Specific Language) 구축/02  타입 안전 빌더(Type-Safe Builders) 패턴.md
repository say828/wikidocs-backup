### 타입 안전 빌더(Type-Safe Builders) 패턴

이제 우리는 DSL의 개념(`18.1`)과 그 핵심 메커니즘(`18.2. 수신 객체가 있는 람다`)을 모두 이해했습니다. **타입 안전 빌더(Type-Safe Builder)** 패턴은 바로 이 두 가지를 조합하여, `person { address { ... } }`와 같이 복잡하고 계층적인 객체를 생성하는 우아하고 구조적인 DSL을 구축하는 최종 디자인 패턴을 의미합니다.

이 패턴의 이름에 '타입 안전(Type-Safe)'이 붙은 이유는, DSL의 구조를 코틀린의 타입 시스템을 통해 컴파일 시점에 강제하기 때문입니다. 즉, 문법적으로 말이 안 되는 구조(예: `address` 블록 안에 또 `address`를 넣거나, `person` 블록 안에 `street` 속성을 직접 넣는 것)를 **컴파일러가 오류로 잡아낼 수 있게** 설계하는 것을 의미합니다.

이는 Jetpack Compose와 Ktor 라우팅이 동작하는 방식과 정확히 동일합니다.

-----

#### 1단계: 데이터 모델 정의 (최종 결과물)

먼저 우리가 DSL을 통해 최종적으로 만들고 싶은 불변(immutable) 데이터 클래스들을 정의합니다.

```kotlin
data class Address(
    val street: String,
    val city: String
)

data class Person(
    val name: String,
    val age: Int,
    val address: Address?
)
```

-----

#### 2단계: 빌더 클래스 정의 (데이터를 임시 저장할 mutable 클래스)

위의 불변 객체를 한 번에 만들기 위해, 우리는 데이터를 임시적으로 담아둘 변경 가능한(`var`) '빌더' 클래스들을 설계합니다. 이 빌더들은 DSL의 "작업대" 역할을 합니다.

```kotlin
// Address 생성을 위한 빌더
class AddressBuilder {
    var street: String = ""
    var city: String = ""

    fun build(): Address = Address(street, city)
}

// Person 생성을 위한 빌더
class PersonBuilder {
    var name: String = ""
    var age: Int = 0
    private var address: Address? = null // 완성된 Address 객체를 저장할 변수

    /**
     * 이것이 바로 중첩 DSL의 핵심입니다.
     * PersonBuilder의 컨텍스트 안에서 address { ... } 람다를 호출할 수 있도록
     * 'address'라는 이름의 메서드를 제공합니다.
     * 이 메서드는 'AddressBuilder'를 수신 객체로 받는 람다를 인자로 받습니다.
     */
    fun address(init: AddressBuilder.() -> Unit) {
        // AddressBuilder를 만들고, 람다로 초기화한 뒤, build()하여 
        // 완성된 Address 객체를 내부 프로퍼티에 저장합니다.
        this.address = AddressBuilder().apply(init).build()
    }

    fun build(): Person = Person(name, age, address)
}
```

-----

#### 3단계: DSL 진입점(Entry Point) 함수 정의

마지막으로, 사용자가 이 빌더를 쉽게 사용할 수 있도록 편리한 최상위 함수(DSL 진입점)를 만들어줍니다. 이 함수가 바로 `18.2`에서 배운 '수신 객체가 있는 람다'를 파라미터로 받는 고차 함수입니다.

```kotlin
fun person(init: PersonBuilder.() -> Unit): Person {
    // 1. PersonBuilder 인스턴스를 생성합니다.
    val builder = PersonBuilder()
    
    // 2. 생성된 빌더 객체를 '수신 객체(this)'로 하여 사용자 람다(init)를 실행합니다.
    builder.init()
    
    // 3. 람다를 통해 모든 설정이 완료된 빌더로 최종 객체를 생성하여 반환합니다.
    return builder.build()
}
```

-----

#### 4단계: 타입 안전 빌더 실행하기

이제 이 세 단계가 조합되어 어떻게 마법 같은 DSL이 완성되는지 확인해 봅시다.

```kotlin
val alice = person {
    // 1. 여기는 'PersonBuilder'의 컨텍스트(this)입니다.
    //    따라서 PersonBuilder의 프로퍼티(name, age)에 바로 접근할 수 있습니다.
    name = "Alice"
    age = 30

    // 2. 'address'는 PersonBuilder에 정의된 메서드입니다.
    address {
        // 3. 이 블록은 'AddressBuilder'의 컨텍스트(this)입니다.
        //    AddressBuilder의 프로퍼티(street, city)에 바로 접근합니다.
        street = "123 Main St"
        city = "Wonderland"

        // age = 50 // 컴파일 오류!
        // AddressBuilder의 컨텍스트 안에는 'age' 프로퍼티가 없습니다.
    }

    // street = "..." // 컴파일 오류!
    // PersonBuilder의 컨텍스트 안에는 'street' 프로퍼티가 없습니다.
}

println(alice)
// 출력: Person(name=Alice, age=30, address=Address(street=123 Main St, city=Wonderland))
```

컴파일러가 `person` 블록 안에서는 `PersonBuilder`의 멤버만, `address` 블록 안에서는 `AddressBuilder`의 멤버만 허용하는 것을 볼 수 있습니다. 이것이 바로 \*\*"타입 안전성"\*\*입니다. 개발자는 정해진 구조(언어)를 따르도록 강제되며, 실수를 할 가능성이 원천적으로 차단됩니다.

이 타입 안전 빌더 패턴은 코틀린 언어의 핵심 기능들(람다, 확장 함수, 수신 객체)이 어떻게 결합하여 복잡한 객체 생성을 위한 우아하고 안전하며 표현력 넘치는 API를 만들어낼 수 있는지 보여주는 완벽한 증거입니다.