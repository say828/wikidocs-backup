### 안전한 호출 연산자(Safe Call Operator): ?.

컴파일러가 Nullable 타입의 멤버에 직접 접근하는 것을 막는다면, 우리는 어떻게 이 변수를 안전하게 사용할 수 있을까요? 컴파일러에게 `null`이 아님을 '증명'하는 가장 일반적이고 우아한 방법이 바로 **안전한 호출 연산자(Safe Call Operator)**, 즉 `?.` 입니다.

`?.` 연산자는 `null` 검사와 멤버 접근을 하나의 동작으로 결합합니다.

**`?.`의 동작 방식**

> "왼쪽에 있는 객체가 `null`이 아니면, 오른쪽의 멤버(프로퍼티 또는 메서드)에 접근하라. 만약 `null`이라면, 더 이상 진행하지 말고 이 표현식 전체를 `null`로 평가하라."

이는 마치 우편함을 열기 전에 📬 안에 편지가 있는지 먼저 살짝 엿보는 것과 같습니다. 편지가 있으면(`not null`) 꺼내서 읽고(`멤버 접근`), 비어 있으면(`null`) 실망하지 않고 그냥 돌아서는(`null` 반환) 것과 같습니다. 이 과정에서 우편함이 비어있다고 해서 소리를 지르는(`NullPointerException` 발생) 일은 절대 없습니다.

-----

#### 기본 사용법

```kotlin
var name: String? = "Kotlin"

// name이 null이 아니므로, .length가 호출되고 결과는 Int? 타입인 6이 됩니다.
val length: Int? = name?.length
println("이름의 길이: $length")

name = null

// name이 null이므로, .length는 호출되지 않고 표현식 전체가 null이 됩니다.
val length2: Int? = name?.length
println("이름의 길이: $length2")
```

**실행 결과:**

```
이름의 길이: 6
이름의 길이: null
```

주목할 점은 `name?.length`의 반환 타입이 `Int`가 아닌 `Int?`라는 것입니다. `name`이 `null`일 경우 표현식 전체가 `null`이 될 수 있으므로, 컴파일러는 결과 타입 역시 Nullable로 추론하여 안전성을 유지합니다.

-----

#### 안전 호출 체이닝 (Safe Call Chaining)

`?.`의 진정한 힘은 여러 호출을 연결(**체이닝, Chaining**)할 때 나타납니다. 객체의 속성이 연쇄적으로 `null`일 가능성이 있을 때, 코드가 놀랍도록 간결해집니다.

```kotlin
// 학생은 소속된 학과가 없을 수도 있고, 학과에는 지도교수가 없을 수도 있다.
class Professor(val name: String)
class Department(val headProfessor: Professor?)
class Student(val department: Department?)

fun getProfessorName(student: Student): String? {
    // student -> department -> headProfessor -> name 순으로 안전하게 접근
    return student.department?.headProfessor?.name
}

val prof = Professor("김교수")
val compSci = Department(prof)
val studentA = Student(compSci)
println(getProfessorName(studentA)) // 출력: 김교수

val studentB = Student(Department(null)) // 지도교수가 없는 학과
println(getProfessorName(studentB)) // 출력: null

val studentC = Student(null) // 소속 학과가 없는 학생
println(getProfessorName(studentC)) // 출력: null
```

만약 `?.`가 없었다면, 이 로직은 `if (student.department != null)` 와 같은 중첩된 `if`문으로 매우 지저분해졌을 것입니다. `?.` 체이닝은 이 중 어느 하나라도 `null`이면, 즉시 전체 표현식을 `null`로 평가하고 안전하게 중단합니다.

-----

#### `?.let` 스코프 함수와 함께 사용하기

`null`이 아닐 경우, 특정 작업을 수행하고 싶을 때 `?.let` 패턴을 매우 유용하게 사용할 수 있습니다. `let`은 자신을 호출한 객체를 인자로 받아 코드 블록을 실행하는 함수이며, `?.`과 결합하면 '객체가 `null`이 아닐 경우에만 이 코드 블록을 실행하라'는 의미가 됩니다.

```kotlin
val name: String? = "Gyeonggi-do, Seongnam-si"

name?.let {
    // 이 블록은 name이 null이 아닐 때만 실행됩니다.
    // 블록 안에서 'it'은 null이 아닌 String 타입으로 스마트 캐스팅됩니다.
    println("이름: $it")
    println("길이: ${it.length}")
    println("대문자: ${it.uppercase()}")
}
```

`?.let`은 `null`이 아닌 값에 대해 여러 연산을 안전하고 깔끔하게 처리할 수 있는 매우 일반적인 코틀린 관용구(Idiom)입니다.

안전한 호출 연산자 `?.`는 코틀린의 널 안정성을 지키는 가장 기본적이고 핵심적인 도구입니다. 위험한 `!!` 연산자 대신 `?.`를 우선적으로 사용하는 습관은 견고하고 안정적인 코틀린 코드를 작성하는 지름길입니다.