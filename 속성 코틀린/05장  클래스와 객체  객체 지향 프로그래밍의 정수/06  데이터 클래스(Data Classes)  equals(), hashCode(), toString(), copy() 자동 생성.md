### 데이터 클래스(Data Classes): equals(), hashCode(), toString(), copy() 자동 생성

프로그래밍을 하다 보면, 복잡한 로직 없이 순수하게 **데이터를 보관하는 목적**으로만 사용되는 클래스를 만들 일이 매우 많습니다. 사용자 정보(User), API 응답(Response) 등을 담는 DTO(Data Transfer Object)가 대표적인 예입니다.

자바에서는 이런 클래스를 만들 때마다 `equals()`, `hashCode()`, `toString()`과 같은 메서드들을 직접 오버라이딩해야 했습니다. 이들은 객체의 동등성 비교나 컬렉션에서의 올바른 동작, 그리고 디버깅 시 유용한 로그 출력을 위해 필수적이지만, 매번 작성하기에는 매우 번거롭고 실수하기도 쉬운 보일러플레이트 코드였습니다.

코틀린은 `data`라는 키워드 하나로 이 모든 고통을 해결합니다.

#### `data` 키워드의 마법

클래스 앞에 `data`라는 변경자 하나만 붙여주면, 코틀린 컴파일러는 주 생성자에 선언된 프로퍼티들을 기반으로 다음 메서드들을 **자동으로 생성**해 줍니다.

  * `equals()`: 객체의 내용(프로퍼티 값)을 비교하여 동등성을 판단합니다.
  * `hashCode()`: `equals()`와 일관성을 가지는 해시 코드를 생성합니다. `HashMap`이나 `HashSet` 같은 컬렉션에서 객체를 올바르게 다루기 위해 필수적입니다.
  * `toString()`: `User(name=Alice, age=30)`와 같이 객체의 내용을 보기 좋게 출력해 줍니다.
  * `componentN()`: 각 프로퍼티에 순서대로 접근할 수 있는 함수들로, '구조 분해 선언'을 가능하게 합니다.
  * `copy()`: 객체를 복사하면서 일부 프로퍼티 값만 변경할 수 있는 매우 유용한 메서드입니다.

#### 일반 클래스 vs. 데이터 클래스

`data` 키워드의 있고 없고의 차이를 코드로 직접 확인해 봅시다.

##### 일반 클래스의 경우

```kotlin
class User(val name: String, val age: Int)

fun main() {
    val user1 = User("Alice", 30)
    val user2 = User("Alice", 30)

    println(user1)          // 출력: User@2f92e0f4 (클래스이름@메모리주소)
    println(user1 == user2) // 출력: false (내용이 같아도 참조(주소)가 다르므로)
}
```

##### 데이터 클래스의 경우

```kotlin
data class DataUser(val name: String, val age: Int)

fun main() {
    val user1 = DataUser("Alice", 30)
    val user2 = DataUser("Alice", 30)

    println(user1)          // 출력: DataUser(name=Alice, age=30) (toString() 자동 생성)
    println(user1 == user2) // 출력: true (equals()가 내용을 비교하도록 자동 생성)
}
```

결과가 완전히 다른 것을 볼 수 있습니다. `data` 키워드 하나만으로 클래스가 우리가 기대했던 대로 '데이터'로서 올바르게 동작하기 시작했습니다.

-----

#### `copy()`: 불변 객체를 우아하게 다루는 법

`copy()` 메서드는 데이터 클래스의 가장 강력한 기능 중 하나입니다. 불변(`val`) 프로퍼티를 가진 객체의 일부 값만 변경하여 새로운 객체를 만들고 싶을 때 매우 유용합니다.

```kotlin
val user1 = DataUser("Alice", 30)

// user1을 복사하되, age 값만 31로 변경하여 새로운 user2 객체를 생성
val user2 = user1.copy(age = 31)

println(user1) // 출력: DataUser(name=Alice, age=30) (원본은 불변)
println(user2) // 출력: DataUser(name=Alice, age=31) (새로운 복사본)
```

`copy()`는 원본 객체의 불변성을 해치지 않으면서 상태를 변경한 새로운 객체를 손쉽게 만들 수 있도록 도와주므로, 안전하고 예측 가능한 프로그래밍 스타일에 크게 기여합니다.

#### 구조 분해 선언 (Destructuring Declaration)

데이터 클래스는 `componentN()` 함수가 자동으로 생성되기 때문에, 객체가 가진 프로퍼티들을 여러 개의 변수로 한 번에 분해하여 할당할 수 있습니다.

```kotlin
val user = DataUser("Bob", 25)

// user 객체를 name과 age 변수로 분해합니다.
// 내부적으로 val name = user.component1() 와 val age = user.component2()가 호출됩니다.
val (name, age) = user

println("이름: $name, 나이: $age") // 출력: 이름: Bob, 나이: 25
```

**데이터 클래스 사용 규칙:**

  * 주 생성자는 하나 이상의 파라미터를 가져야 합니다.
  * 주 생성자의 모든 파라미터는 `val` 또는 `var`로 선언되어야 합니다.
  * 데이터 클래스는 `abstract`, `open`, `sealed`, `inner` 클래스가 될 수 없습니다.

`data class`는 코틀린이 개발자의 생산성을 얼마나 중요하게 생각하는지를 보여주는 대표적인 사례입니다. 단순히 코드를 줄여주는 것을 넘어, 객체가 데이터로서 올바르게 동작하도록 보장하고 불변성 유지를 돕는 필수적인 도구입니다.