## 01\. Null-Safety와 `@Column(nullable = false)`의 완벽한 조화

코틀린 언어의 가장 위대한 성취 중 하나는 바로 \*\*Null-Safety(널 안전성)\*\*다. 코틀린은 컴파일 시점에 `NullPointerException`(NPE)의 발생 가능성을 원천적으로 차단하여, '10억 달러의 실수'라고 불렸던 `null` 참조의 위험으로부터 개발자를 해방시켰다.

이 강력한 언어적 특성은 데이터베이스의 `NOT NULL` 제약조건과 만났을 때 완벽한 시너지를 발휘하며, 데이터의 무결성을 애플리케이션 계층부터 데이터베이스 계층까지 관통하여 보장하는 견고한 방어선을 구축한다. 🛡️

-----

### **두 세계의 만남: `String` vs. `String?`**

  * **코틀린의 세계**: 타입 시스템을 통해 `String`(절대 `null`이 될 수 없는 타입)과 `String?`(null이 될 수 있는 타입)을 명확하게 구분한다. `String` 타입의 변수에 `null`을 할당하려고 하면 컴파일 에러가 발생한다.

  * **데이터베이스의 세계**: 테이블의 각 컬럼에 `NOT NULL` 제약조건을 설정하여 `null` 값의 저장을 막을 수 있다.

JPA는 이 두 세계를 잇는 다리 역할을 한다. 우리는 코틀린의 타입 시스템과 JPA의 `@Column(nullable = false)` 어노테이션을 함께 사용하여, 두 세계의 규칙을 완벽하게 동기화할 수 있다.

**완벽한 조화를 이룬 코드**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long = 0,

    // 코틀린 타입도 non-null, DB 제약조건도 NOT NULL
    @Column(nullable = false)
    var username: String,
    
    // 코틀린 타입도 nullable, DB 제약조건도 nullable (기본값)
    var description: String? = null
)
```

**`username` 필드를 살펴보자.**

1.  **애플리케이션 계층**: 코틀린 컴파일러는 `username` 필드에 `null`이 할당되는 것을 절대 허용하지 않는다. 이 `Member` 객체를 사용하는 모든 코드는 `username`이 항상 존재한다고 믿고 안전하게 `member.username.length` 와 같은 코드를 사용할 수 있다.
2.  **데이터베이스 계층**: `@Column(nullable = false)` 설정으로 인해, `MEMBER` 테이블의 `username` 컬럼에는 `NOT NULL` 제약조건이 걸린다. 데이터베이스 스스로가 `username`에 `null` 값이 저장되는 것을 막아 데이터의 무결성을 보장한다.

이처럼 두 계층에서 동일한 규칙을 강제함으로써, 우리는 `null`로 인해 발생할 수 있는 거의 모든 종류의 버그를 원천적으로 차단할 수 있다.

-----

> **결론: Non-null 프로퍼티에는 항상 `nullable = false`를 붙여라.**
>
> 코틀린으로 JPA 엔티티를 설계할 때, 특별한 이유가 없는 한 `var description: String? = null` 처럼 필드를 nullable로 선언하는 것을 피해야 한다. 비즈니스 규칙상 반드시 값이 존재해야 하는 필드라면, `var username: String` 처럼 non-null 타입으로 선언하고, 반드시 `@Column(nullable = false)`를 함께 명시하는 것을 습관화해야 한다.
>
> 이것은 단순히 코드를 깔끔하게 만드는 것을 넘어, 애플리케이션의 안정성을 극대화하고, 예측 불가능한 런타임 오류를 컴파일 타임의 견고한 방어벽으로 대체하는 매우 중요한 설계 원칙이다.

이제 코틀린의 Null-Safety를 정복했으니, 다음으로는 JPA의 지연 로딩(Lazy Loading)과 깊은 관련이 있는 코틀린의 `open` 키워드에 대해 알아보자.