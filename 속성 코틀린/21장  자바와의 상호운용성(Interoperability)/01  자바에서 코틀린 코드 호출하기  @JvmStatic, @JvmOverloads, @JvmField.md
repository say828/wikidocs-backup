### 자바에서 코틀린 코드 호출하기: @JvmStatic, @JvmOverloads, @JvmField

코틀린에서 자바를 호출하는 것이 거의 마법처럼 매끄러웠다면, 반대로 자바에서 코틀린 코드를 호출하는 것은 약간의 '번역'이 필요합니다. 자바 언어는 코틀린이 가진 프로퍼티, 최상위 함수, 기본 인자 같은 현대적인 기능들을 이해하지 못하기 때문입니다.

코틀린 컴파일러는 이 모든 기능을 JVM 바이트코드로 변환하지만, 그 결과물이 자바 개발자에게는 다소 생소하거나 불편할 수 있습니다. 코틀린은 이러한 마찰을 줄이고 자바 개발자에게도 친숙한 API를 제공하기 위해, 바이트코드가 생성되는 방식을 제어하는 특별한 **어노테이션**들을 제공합니다.

-----

#### 1\. 프로퍼티(Properties)와 `@JvmField`

자바에서 코틀린 클래스의 프로퍼티를 호출하면, 코틀린 컴파일러는 `private` 필드와 그에 대한 `getter/setter`를 자동으로 생성해 줍니다. 이는 코틀린이 자바의 `get`/`set`을 프로퍼티로 변환해 준 것과 정반대의 과정입니다.

```kotlin
// Kotlin Code
class User(var name: String)
```

```java
// Java Code
User user = new User("Kotlin");
String name = user.getName(); // getName()으로 호출해야 함
user.setName("Java");        // setName()으로 호출해야 함
```

대부분의 경우 이는 잘 동작하지만, Jackson이나 JPA처럼 리플렉션을 통해 필드에 직접 접근해야 하는 일부 자바 프레임워크에서는 문제가 될 수 있습니다.

**해결책: `@JvmField`**
프로퍼티에 `@JvmField` 어노테이션을 붙이면, 코틀린 컴파일러는 `getter/setter` 생성을 건너뛰고, 해당 프로퍼티를 자바의 **`public` 필드**로 직접 노출시킵니다.

```kotlin
// Kotlin Code
class User(@JvmField var name: String) // public 필드로 노출
```

```java
// Java Code
User user = new User("Kotlin");
String name = user.name; // 필드로 직접 접근 가능!
user.name = "Java";
```

-----

#### 2\. 동반 객체(Companion Objects)와 `@JvmStatic`

코틀린 클래스 내의 `companion object`는 클래스 인스턴스와 무관하게 공유되는 정적(static) 멤버의 집합입니다. 하지만 자바의 눈에 `companion object`는 그저 `Companion`이라는 이름을 가진 싱글턴 객체일 뿐입니다.

```kotlin
// Kotlin Code
class MyClass {
    companion object {
        fun create(): MyClass = MyClass()
    }
}
```

```java
// Java Code (불편한 방식)
// MyClass.create()가 아니라, 'Companion' 객체를 통해 접근해야 함
MyClass obj = MyClass.Companion.create();
```

자바 개발자는 `MyClass.create()`와 같은 정적 메서드 호출을 기대합니다.

**해결책: `@JvmStatic`**
`companion object` 내부의 함수나 프로퍼티에 `@JvmStatic`을 붙이면, 코틀린 컴파일러는 두 가지 버전의 코드를 모두 생성해 줍니다. 하나는 코틀린을 위한 `Companion` 객체의 멤버이고, 다른 하나는 **자바를 위한 진정한 `public static` 멤버**입니다.

```kotlin
// Kotlin Code
class MyClass {
    companion object {
        @JvmStatic // 자바용 static 메서드를 추가로 생성
        fun create(): MyClass = MyClass()
    }
}
```

```java
// Java Code (관용적인 방식)
// 이제 자바에서 기대하는 방식대로 정적 메서드 호출이 가능합니다.
MyClass obj = MyClass.create(); 
```

최상위 함수(Top-level function) 역시 `Utils.kt` 파일에 선언하면 `UtilsKt.class`의 정적 메서드로 컴파일되는데, 이때도 `@JvmStatic`이나 `@JvmName` 등이 유용하게 사용될 수 있습니다.

-----

#### 3\. 기본 인자(Default Arguments)와 `@JvmOverloads`

코틀린의 강력한 기능인 기본 인자와 이름 있는 인자는 자바에 존재하지 않는 개념입니다.

```kotlin
// Kotlin Code
fun log(message: String, level: String = "INFO", timestamp: Long = System.currentTimeMillis()) {
    // ...
}
```

자바 컴파일러는 이 코드를 보고 오직 **시그니처가 완전한 하나의 메서드**만 인식합니다: `log(String, String, Long)`. 자바에서는 `log("My Message")`와 같이 하나의 인자만 전달하는 것이 불가능합니다.

**해결책: `@JvmOverloads`**
함수나 생성자에 `@JvmOverloads` 어노테이션을 붙이면, 코틀린 컴파일러가 기본 인자를 사용하여 \*\*여러 버전의 오버로딩된 메서드(Overloaded methods)\*\*를 자바용으로 자동 생성해 줍니다.

```kotlin
// Kotlin Code
class Logger {
    @JvmOverloads // 이 어노테이션 하나로...
    fun log(message: String, level: String = "INFO", timestamp: Long = System.currentTimeMillis()) {
        // ...
    }
}
```

**컴파일러가 생성하는 자바용 메서드들 (개념적으로):**

1.  `public void log(String message, String level, long timestamp)` (모든 인자)
2.  `public void log(String message, String level)` (timestamp에 기본값 사용)
3.  `public void log(String message)` (level과 timestamp에 기본값 사용)

<!-- end list -->

```java
// Java Code
Logger logger = new Logger();
logger.log("Message 1"); // OK
logger.log("Message 2", "DEBUG"); // OK
logger.log("Message 3", "ERROR", 12345L); // OK
```

이 세 가지 어노테이션(`@JvmField`, `@JvmStatic`, `@JvmOverloads`)은 코틀린 API를 자바 개발자에게 더 친숙하고 관용적인 방식으로 노출시켜주는 필수적인 '번역 도구'입니다.