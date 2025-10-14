### 어노테이션 클래스(Annotation Classes)

\*\*어노테이션(Annotation)\*\*은 코드 자체의 일부라기보다는, 코드에 대한 \*\*부가 정보(Metadata)\*\*를 붙이기 위해 사용되는 특별한 문법 요소입니다. 어노테이션은 직접적인 로직을 수행하지 않지만, 컴파일러나 다른 프로그램(프레임워크, 라이브러리 등)이 이 정보를 읽고 특별한 방식으로 코드를 처리하도록 만들 수 있습니다.

🏷️ **비유:** 어노테이션은 코드에 붙이는 '꼬리표'나 '스티커'와 같습니다. `@Deprecated`라는 스티커는 "이 코드는 낡았으니 사용하지 마세요\!"라는 정보를, `@Test`라는 스티커는 "이 함수는 테스트용입니다\!"라는 정보를 알려줍니다.

-----

#### 어노테이션 사용하기

코틀린에서는 이미 표준 라이브러리나 JUnit과 같은 유명 프레임워크에 정의된 다양한 어노테이션을 `@` 기호와 함께 사용합니다.

  * **`@Deprecated`**: 더 이상 사용되지 않는 클래스나 함수를 표시합니다. IDE는 이 코드를 사용하는 곳에 취소선을 표시하고, 더 나은 대안을 제시해 줄 수 있습니다.

    ```kotlin
    @Deprecated("더 이상 사용하지 않습니다. newFunction()으로 대체하세요.", ReplaceWith("newFunction()"))
    fun oldFunction() { /* ... */ }
    ```

  * **`@JvmStatic`**: 동반 객체의 멤버를 자바 코드에서 진짜 `static` 멤버처럼 호출할 수 있도록 해줍니다.

  * **`@Test`**: JUnit과 같은 테스트 프레임워크에게 이 함수가 테스트 케이스임을 알려줍니다.

-----

#### 직접 어노테이션 만들기: `annotation class`

코틀린에서는 `annotation class` 키워드를 사용하여 우리만의 커스텀 어노테이션을 직접 만들 수 있습니다.

##### 기본 어노테이션

```kotlin
// 가장 간단한 형태의 어노테이션 선언
annotation class MyTest
```

##### 파라미터가 있는 어노테이션

어노테이션은 클래스의 주 생성자처럼 파라미터를 가질 수 있어, 더 구체적인 정보를 전달할 수 있습니다.

```kotlin
// JSON 직렬화 시 사용할 키 이름을 지정하는 어노테이션
annotation class JsonName(val name: String)
```

이제 이 어노테이션을 사용하여 클래스 프로퍼티에 추가 정보를 붙일 수 있습니다.

```kotlin
class User(
    @JsonName("user_name") // 이 프로퍼티는 JSON에서 "user_name"이라는 키로 변환될 것이다.
    val name: String,
    
    val age: Int
)
```

이후 Jackson이나 Gson 같은 라이브러리는 리플렉션을 통해 `@JsonName` 어노테이션을 읽고, `name` 프로퍼티를 "name"이 아닌 "user\_name"으로 직렬화하게 됩니다.

-----

#### 어노테이션 사용 대상 지정 (Use-site Targets)

하나의 코드 요소에는 어노테이션을 붙일 수 있는 위치가 여러 개일 수 있습니다. 예를 들어, 클래스의 프로퍼티에는 프로퍼티 자체, getter, setter, 생성자 파라미터 등이 있습니다.

코틀린은 어노테이션이 정확히 어디에 적용되어야 하는지를 명시하기 위해 **사용 대상(Use-site Target)** 문법을 제공합니다.

```kotlin
class User(
    // @param: 어노테이션을 주 생성자의 파라미터에만 적용
    @param:JsonName("user_id") val id: String,

    // @get: 어노테이션을 프로퍼티의 getter에만 적용
    @get:JsonIgnore val internalCode: String 
)
```

주요 사용 대상은 `@property`, `@field`, `@get`, `@set`, `@param` 등이 있습니다.

#### 메타 어노테이션

어노테이션을 정의할 때, 그 어노테이션 자체의 동작을 제어하는 어노테이션을 붙일 수 있으며, 이를 **메타 어노테이션**이라고 합니다.

  * **`@Target`**: 이 어노테이션을 붙일 수 있는 코드 요소의 종류를 제한합니다. (예: `AnnotationTarget.CLASS`, `FUNCTION`, `PROPERTY`)
  * **`@Retention`**: 어노테이션 정보를 언제까지 유지할지를 결정합니다. (`AnnotationRetention.SOURCE`, `BINARY`, `RUNTIME`)
  * **`@Repeatable`**: 같은 요소에 동일한 어노테이션을 여러 번 붙일 수 있도록 허용합니다.
  * **`@MustBeDocumented`**: API 문서에 이 어노테이션이 포함되도록 합니다.

<!-- end list -->

```kotlin
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION) // 클래스와 함수에만 사용 가능
@Retention(AnnotationRetention.RUNTIME) // 런타임에도 어노테이션 정보를 유지
annotation class MyFrameworkAnnotation
```

어노테이션은 코드에 선언적인 방식으로 메타데이터를 추가하는 강력한 메타프로그래밍 도구입니다. 이는 스프링(Spring), JUnit, Room 등 수많은 현대적인 프레임워크와 라이브러리가 보일러플레이트 코드를 줄이고 강력한 기능을 제공하는 핵심 원리입니다.