## Null 안전성(Nullable/Platform Types)의 실용적 이해: JPA와 @Entity에서의 함정 피하기

00장에서 코틀린의 가장 강력한 무기 중 하나로 'Null 안전성'을 꼽았습니다. `String`과 `String?`을 컴파일 시점에 명확히 구분하여 `NullPointerException` (NPE)을 원천 차단하는 기능입니다.

하지만, 우리가 스프링 부트에서 JPA(Java Persistence API)를 Hibernate 구현체와 함께 사용하는 순간, 이 '완벽한' 보증은 깨지기 시작합니다. 바로 코틀린이 아닌 **자바(Java)로 작성된 라이브러리와의 상호작용** 때문입니다.

-----

### Platform Types: 보증이 깨지는 회색 지대

JPA와 Hibernate는 자바 기반입니다. 자바의 모든 참조 타입(예: `String`)은 `null`이 될 수도, 안 될 수도 있습니다. 코틀린이 이런 자바 코드를 호출할 때, 컴파일러는 해당 타입의 Nullable 여부를 100% 확신할 수 없습니다.

이때 등장하는 것이 \*\*'플랫폼 타입(Platform Type)'\*\*이며, 코틀린 IDE에서는 `String!`와 같이 `!`로 표시됩니다.

`String!`는 "이것은 `String`일 수도 있고 `String?`일 수도 있으니, 개발자 네가 알아서 처리해"라는 의미입니다. 만약 개발자가 이것을 `String` (Non-Null)으로 간주하고 받았는데, 실제 런타임에 Hibernate가 `null`을 반환하면, 코틀린 코드임에도 불구하고 **NPE가 발생합니다.**

-----

### 함정 1: `@Entity`에 `data class`를 쓰지 마라

`data class`는 DTO에는 완벽하지만, JPA의 `@Entity`로 사용하기에는 치명적인 단점들을 가지고 있습니다.

1.  **No-Arg Constructor (기본 생성자) 문제:** JPA 스펙상 `@Entity` 클래스는 `public` 또는 `protected` 기본 생성자(인자가 없는 생성자)를 반드시 가져야 합니다. Hibernate가 DB에서 데이터를 조회하여 객체로 만들 때 이 생성자를 사용하기 때문입니다.
    하지만 `data class`는 모든 프로퍼티를 인자로 받는 '주 생성자'를 기반으로 하며, 기본 생성자를 자동으로 만들지 않습니다. (물론 `kotlin-jpa` 플러그인이 이 문제를 일부 해결해 주지만, 근본적인 문제가 남습니다.)

2.  **`equals()`와 `hashCode()` 문제:** `data class`가 자동 생성하는 `equals()`와 `hashCode()`는 객체의 '데이터'만을 비교합니다. 하지만 Hibernate는 '지연 로딩(Lazy Loading)'을 위해 프록시(Proxy) 객체를 생성하는 경우가 많습니다.
    이때, 아직 로드되지 않은 연관관계 필드에 접근하려다 `LazyInitializationException`이 터지거나, 프록시 객체와 실제 객체 간의 `equals()` 비교가 실패하는 등, 예측 불가능한 문제를 일으킬 수 있습니다.

3.  **`toString()` 문제:** `toString()` 역시 지연 로딩된 필드에 접근하려다 예외를 터뜨릴 수 있습니다.

**결론: `@Entity`에는 `data class`가 아닌 일반 `class`를 사용해야 합니다.**

-----

### 함정 2: `lateinit` vs `default value` (프로퍼티 초기화)

`@Entity`에 일반 `class`를 쓰기로 결정해도, 코틀린의 'Non-Null' 타입 규칙은 여전히 JPA의 '기본 생성자' 요구사항과 충돌합니다.

```kotlin
@Entity
class Member(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null, // 1. ID는 생성 시점엔 null이어야 하므로 Nullable
    
    var name: String,     // 2. DB는 NOT NULL 컬럼. 하지만...
    var age: Int          // 3. DB는 NOT NULL 컬럼. 하지만...
)
// 위 코드는 컴파일 에러가 발생합니다!
// "Property 'name' must be initialized or be abstract"
```

JPA가 기본 생성자로 `Member()`를 호출할 때, `name`과 `age`는 `String`과 `Int` 타입이므로 `null`이 될 수 없어 반드시 초기화가 되어야 합니다. 이 딜레마를 해결하는 방법은 두 가지이며, 각각 장단점이 명확합니다.

#### 방법 A: `lateinit var` (지연 초기화) - 권장하지만 주의 필요

`lateinit` 키워드는 "지금은 초기화하지 않지만, 내가 나중에 사용하기 전까지는 반드시 초기화할 것을 컴파일러에게 약속할게"라는 의미입니다.

```kotlin
@Entity
class Member(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,
    
    @Column(nullable = false)
    lateinit var name: String, // 'var'만 사용 가능
    
    @Column(nullable = false)
    var age: Int = 0 // 'Int' 같은 Primitive Type은 lateinit 불가
)
```

  * **장점:** 코드가 깔끔하고, `name` 프로퍼티가 Non-Null(`String`)임을 명확히 드러냅니다.
  * **단점:** `lateinit`의 '약속'을 어기고, Hibernate가 값을 채우기 전에 `name`에 접근하면(예: 생성 직후 로직 실수), `UninitializedPropertyAccessException`라는 또 다른 런타임 예외를 만납니다.
  * **참고:** `Int`, `Long`, `Boolean` 같은 Primitive 타입에는 `lateinit`을 쓸 수 없습니다. 따라서 `age`는 `0`과 같은 기본값을 가져야 합니다.

#### 방법 B: `default value` (기본값 할당) - 가장 안전한 방법

모든 Non-Null 프로퍼티에 '논리적인' 기본값을 할당하는 것입니다.

```kotlin
@Entity
class Member(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,
    
    @Column(nullable = false)
    var name: String = "", // 기본값으로 빈 문자열 할당
    
    @Column(nullable = false)
    var age: Int = 0,
    
    var address: String? = null // Nullable 컬럼은 null로 초기화
)
```

  * **장점:** 가장 안전합니다. JPA가 기본 생성자를 호출해도 모든 필드가 즉시 초기화되며, 런타임 예외가 발생할 여지가 없습니다.
  * **단점:** `name = ""` 처럼, 비즈니스 로직상 의미 없는 기본값이 코드에 포함될 수 있습니다.

**결론:** 이 책에서는 \*\*`lateinit var`\*\*와 \*\*`default value`\*\*를 상황에 맞게 혼용하되, `Int`나 `Long`처럼 기본값(예: `0L`)이 명확한 경우는 '기본값 할당'을, `String`처럼 애매한 경우는 `lateinit`을 사용하여 Non-Null 제약을 명시적으로 드러내는 방식을 택하겠습니다. 그리고 Nullable이 허용되는 필드(`address`)는 반드시 `String?` 타입과 `null` 기본값을 사용합니다.