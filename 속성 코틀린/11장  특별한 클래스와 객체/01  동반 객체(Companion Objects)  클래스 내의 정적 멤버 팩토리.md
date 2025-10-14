### 동반 객체(Companion Objects): 클래스 내의 정적 멤버 팩토리

자바 개발자에게 `static` 키워드는 매우 친숙합니다. `static` 멤버는 클래스의 특정 인스턴스에 속하지 않고, 클래스 자체에 소속되어 프로그램 전체에서 공유되는 변수나 메서드를 만들 때 사용됩니다. `Math.PI`와 같은 상수나, `Integer.parseInt()`와 같은 유틸리티 메서드가 대표적인 예입니다.

하지만 코틀린에는 `static` 키워드가 없습니다. 이는 코틀린이 순수한 객체 지향 언어를 지향하며, 모든 것이 객체라는 원칙을 지키기 때문입니다. 그렇다면 코틀린에서는 클래스 수준의 멤버를 어떻게 표현할까요? 그 해답이 바로 \*\*동반 객체(Companion Object)\*\*입니다.

-----

#### `companion object`란?

**동반 객체**는 \*\*클래스 내부에 선언되는 특별한 `object`\*\*입니다. 모든 클래스는 단 하나의 동반 객체만 가질 수 있으며, 이 객체는 자신을 감싸는 클래스와 '동반'하며 평생을 함께하는 유일한 동반자 짝과 같습니다.

동반 객체 내부에 선언된 프로퍼티나 메서드는 외부에서 **클래스 이름을 통해 직접 접근**할 수 있어, 자바의 `static` 멤버와 거의 동일하게 동작합니다.

**비유:** 동반 객체는 '대학교(`Class`)의 중앙 행정실(`Companion Object`)'과 같습니다. 행정실은 대학교 전체에 단 하나만 존재하며, '신입생 입학 처리(`팩토리 메서드`)'나 '학교 설립 연도(`상수`)'와 같이 특정 학생(`인스턴스`)이 아닌 대학교 전체와 관련된 업무를 담당합니다.

```kotlin
class Circle(val radius: Double) {
    
    // Circle 클래스의 동반 객체
    companion object {
        const val PI = 3.14159 // 상수 선언
        
        fun newInstance(radius: Double): Circle {
            return Circle(radius)
        }
    }

    fun getArea(): Double {
        return radius * radius * PI // 클래스 내부에서는 동반 객체 멤버에 바로 접근 가능
    }
}

fun main() {
    // 클래스 이름을 통해 동반 객체 멤버에 직접 접근 (자바의 static처럼)
    val circle = Circle.newInstance(5.0)
    println("원의 넓이: ${circle.getArea()}")
    println("원주율: ${Circle.PI}")
}
```

> **`const val`**: `const`는 컴파일 시점에 값이 결정되는 진정한 상수를 의미합니다. `val`보다 더 엄격한 불변성을 가지며, 원시 타입과 `String`에만 사용할 수 있습니다. 동반 객체 내에서 상수를 정의할 때 주로 사용됩니다.

-----

#### 팩토리 메서드 패턴의 핵심

동반 객체의 가장 중요하고 일반적인 용도는 **팩토리 메서드(Factory Method)** 패턴을 구현하는 것입니다. 생성자를 `private`으로 만들어 외부에서 직접 객체를 생성하는 것을 막고, 대신 동반 객체 내에 특정 목적을 가진 생성 함수들을 `public`으로 제공하는 방식입니다.

**장점:**

1.  **생성자의 역할을 명확히 하는 이름**을 가질 수 있습니다. (`User.createWithEmail(...)`, `User.createGuest()`)
2.  객체 생성 전에 입력값 검증, 캐싱 등 **복잡한 전처리 로직**을 넣을 수 있습니다.
3.  실제로는 하위 클래스의 객체를 반환하는 등, **생성 로직을 유연하게 제어**할 수 있습니다.

<!-- end list -->

```kotlin
class User private constructor(val nickname: String) { // 주 생성자를 private으로!

    companion object {
        fun newSubscribingUser(email: String): User {
            // 이메일에서 아이디 부분을 닉네임으로 사용
            val nickname = email.substringBefore('@')
            return User(nickname)
        }
        
        fun newGuestUser(): User {
            return User("Guest")
        }
    }
}

fun main() {
    // val user1 = User("alice") // 컴파일 오류! 생성자가 private이라 직접 호출 불가

    val subscribingUser = User.newSubscribingUser("alice@example.com")
    val guestUser = User.newGuestUser()
    
    println(subscribingUser.nickname) // 출력: alice
    println(guestUser.nickname)       // 출력: Guest
}
```

#### `object`와의 차이점

일반 `object`는 독립적인 싱글턴이지만, `companion object`는 특정 클래스에 종속되어 그 클래스의 `private` 멤버에도 접근할 수 있다는 강력한 특징을 가집니다.

동반 객체는 단순히 `static`을 흉내 내는 것을 넘어, 객체로서 인터페이스를 구현하는 등 훨씬 더 강력하고 유연한 기능을 제공합니다. 이는 `static`이라는 절차지향적 유산을 객체 지향의 패러다임 안으로 우아하게 통합한 코틀린의 설계 철학을 보여줍니다.