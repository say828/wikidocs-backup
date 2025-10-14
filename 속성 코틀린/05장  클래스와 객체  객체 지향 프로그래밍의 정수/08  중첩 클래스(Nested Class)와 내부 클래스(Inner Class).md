### 중첩 클래스(Nested Class)와 내부 클래스(Inner Class)

때로는 어떤 클래스가 오직 다른 특정 클래스 안에서만 의미를 가지거나, 논리적으로 강하게 연결되어 있을 때가 있습니다. 이럴 때 클래스 내부에 또 다른 클래스를 정의하여 코드의 구조를 명확하게 하고 캡슐화를 강화할 수 있습니다.

코틀린은 클래스 안에 클래스를 정의하는 두 가지 방법을 제공합니다: **중첩 클래스**와 **내부 클래스**. 둘은 문법적으로 비슷해 보이지만, 외부 클래스의 인스턴스와의 관계에 있어 결정적인 차이를 가집니다.

-----

#### 중첩 클래스 (Nested Class): 독립적인 이웃

**중첩 클래스**는 다른 클래스 내부에 선언된 클래스이며, 코틀린에서는 **기본(default)** 방식입니다.

**핵심 특징:** 중첩 클래스는 자신을 감싸는 외부 클래스(Outer Class)의 **인스턴스에 대한 참조를 가지지 않습니다.** 따라서 외부 클래스의 프로퍼티나 메서드에 접근할 수 없습니다. 이는 자바의 `정적 중첩 클래스(static nested class)`와 동일합니다.

**비유:** 중첩 클래스는 '큰 건물(`Outer`) 안에 있는 독립된 사무실(`Nested`)'과 같습니다. 사무실이 건물 안에 있다는 소속 관계는 있지만, 사무실 자체가 특정 건물주(`Outer`의 인스턴스)에게 종속되어 있지는 않습니다.

```kotlin
class Computer { // 외부 클래스
    private val serialNumber = "SN12345"

    class Processor { // 중첩 클래스
        fun getCoreInfo() {
            // println(serialNumber) // 컴파일 오류!
            // 외부 클래스 Computer의 멤버에 접근할 수 없다.
            println("CPU: 8 Cores")
        }
    }
}

fun main() {
    // 중첩 클래스의 인스턴스를 만들 때는 외부 클래스의 이름을 통해 접근합니다.
    val processor = Computer.Processor()
    processor.getCoreInfo()
}
```

**언제 사용할까?** ➡️ 두 클래스가 논리적으로 강하게 묶여있어 함께 관리하고 싶지만, 내부 클래스가 외부 클래스의 상태(인스턴스)와는 독립적으로 동작할 때 사용합니다. (예: `LinkedList` 클래스 내부의 `Node` 클래스)

-----

#### 내부 클래스 (Inner Class): 끈끈한 가족

**내부 클래스**는 `inner` 키워드를 사용하여 선언된 중첩 클래스입니다.

**핵심 특징:** 내부 클래스는 자신을 감싸는 외부 클래스의 **인스턴스에 대한 참조를 가집니다.** 따라서 외부 클래스의 프로퍼티나 메서드에 **접근할 수 있습니다.** 이는 자바의 `비정적 내부 클래스(non-static inner class)`와 동일합니다.

**비유:** 내부 클래스는 '주택(`Outer`) 안에 있는 방(`Inner`)'과 같습니다. 방은 특정 주택의 일부이며, 방 안에서는 거실에 있는 TV(`Outer`의 프로퍼티)를 켜거나 끌 수 있습니다.

```kotlin
class Computer { // 외부 클래스
    private val serialNumber = "SN12345"

    inner class HardDrive { // inner 키워드로 내부 클래스 선언
        fun getSerialNumber() {
            // 외부 클래스의 private 멤버에도 접근 가능!
            println("This hard drive belongs to computer: $serialNumber") 
        }
    }
}

fun main() {
    // 내부 클래스의 인스턴스를 만들려면, 먼저 외부 클래스의 인스턴스가 필요합니다.
    val myComputer = Computer()
    
    // 외부 클래스의 인스턴스를 통해 내부 클래스의 인스턴스를 생성합니다.
    val myHardDrive = myComputer.HardDrive() 
    
    myHardDrive.getSerialNumber()
}
```

**언제 사용할까?** ➡️ 내부 클래스의 존재와 행동이 특정 외부 클래스 인스턴스의 상태에 강하게 의존할 때 사용합니다. (예: 컬렉션 클래스 내부의 `Iterator` 구현체)

-----

#### 핵심 차이점 요약

| 구분 | 중첩 클래스 (Nested) | 내부 클래스 (Inner) |
| :--- | :--- | :--- |
| **선언** | `class Outer { class Nested }` | `class Outer { inner class Inner }` |
| **외부 참조** | **없음** (Java의 static nested class) | **있음** (Java의 non-static inner class) |
| **외부 멤버 접근**| **불가능** | **가능** |
| **인스턴스 생성**| `Outer.Nested()` | `outerInstance.Inner()` |

코틀린은 자바와 달리, 실수로 메모리 누수를 유발할 수 있는 내부 클래스 대신, 더 안전한 중첩 클래스를 기본값으로 채택했습니다. 외부 클래스에 대한 참조가 꼭 필요한 경우에만 `inner` 키워드를 명시하도록 하여 개발자의 의도를 명확하게 하고 실수를 방지합니다.

-----

이것으로 객체 지향의 심장인 클래스와 객체에 대한 5장의 여정을 마칩니다. 클래스 정의의 기초부터 생성자, 상속, 추상화, 캡슐화, 그리고 특별한 목적을 가진 다양한 클래스 유형까지, 당신은 이제 코틀린으로 현실 세계의 복잡한 관계와 개념을 모델링할 수 있는 강력한 도구를 갖추게 되었습니다.