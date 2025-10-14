## 03\. @Enumerated와 @Temporal(LocalDate/Time): 단순 타입을 넘어선 매핑

지금까지 `String`, `Long`, `BigDecimal` 과 같은 기본적인 타입들을 매핑하는 방법을 배웠다. 하지만 실제 애플리케이션에서는 이보다 더 복잡한 타입들, 특히 열거형(Enum)이나 날짜/시간을 다루는 일이 훨씬 많다. JPA는 이러한 타입들을 위한 특별한 매핑 전략을 제공한다.

### **열거형(Enum) 매핑: `@Enumerated`**

애플리케이션에서는 데이터의 상태나 타입을 명확하게 구분하기 위해 열거형을 자주 사용한다. 예를 들어, 회원의 역할을 `USER`와 `ADMIN`으로 구분하는 `Role`이라는 Enum이 있다고 가정해보자.

```kotlin
enum class Role {
    USER, ADMIN
}
```

이 `Role` 타입을 엔티티의 필드로 사용하려면 데이터베이스에 어떤 형태로 저장해야 할까? 데이터베이스에는 `Enum`이라는 타입이 없으므로, 결국 숫자나 문자열로 변환해서 저장해야 한다. `@Enumerated` 어노테이션은 바로 이 변환 방식을 지정하는 역할을 한다.

`@Enumerated`는 두 가지 전략(`EnumType`)을 제공하는데, 이 둘의 차이를 이해하는 것은 매우 중요하다. **잘못된 선택은 미래에 끔찍한 데이터 재앙을 불러올 수 있다.**

**1. `EnumType.ORDINAL`**

  * **동작 방식**: Enum의 순서(0부터 시작하는 index)를 데이터베이스에 저장한다.
  * **예시**: `Role.USER`는 `0`, `Role.ADMIN`은 `1`로 저장된다.
  * **치명적인 단점**: \*\*절대로 사용해서는 안 되는 안티패턴(Anti-pattern)\*\*이다. 만약 미래에 `Role`의 종류가 추가되어 순서가 바뀐다면(예: `enum class Role { GUEST, USER, ADMIN }`), 기존에 `0`으로 저장되었던 모든 `USER`는 순식간에 `GUEST`로 둔갑해버린다. 데이터의 정합성이 완전히 깨지는 것이다.

**2. `EnumType.STRING`**

  * **동작 방식**: Enum의 이름 자체를 문자열로 데이터베이스에 저장한다.
  * **예시**: `Role.USER`는 `"USER"`, `Role.ADMIN`은 `"ADMIN"`으로 저장된다.
  * **장점**: 데이터베이스에 저장된 값이 명확하게 읽히고, Enum의 순서가 바뀌거나 중간에 새로운 값이 추가되어도 기존 데이터에 아무런 영향을 주지 않는다. ORDINAL 방식보다 저장 공간을 조금 더 차지하지만, 데이터의 안정성을 생각하면 이는 아주 사소한 비용이다.

<!-- end list -->

```kotlin
@Entity
class Member(
    // ...
    @Enumerated(EnumType.STRING) // 반드시 STRING을 사용하자.
    @Column(nullable = false, length = 10)
    var role: Role
)
```

> **결론: Enum 타입을 매핑할 때는 아무것도 묻지도 따지지도 말고 `@Enumerated(EnumType.STRING)`을 사용해야 한다.** 이것은 선택이 아닌 필수 규칙이다.

-----

### **날짜/시간 매핑: 안녕 `@Temporal`, 반가워 `LocalDate`**

자바 8 이전, 날짜와 시간을 다루는 `java.util.Date`와 `java.util.Calendar`는 여러 문제점을 가진 클래스였다. 이 낡은 타입들을 데이터베이스의 `DATE`(날짜), `TIME`(시간), `TIMESTAMP`(날짜+시간) 타입과 매핑하기 위해 JPA는 `@Temporal` 어노테이션을 제공했다.

```java
// 과거의 방식 (Java) - 이제는 사용할 필요 없다.
@Temporal(TemporalType.TIMESTAMP)
private Date createdDate;
```

하지만 자바 8에서 스레드에 안전하고 불변이며, 훨씬 직관적인 `java.time` 패키지(`LocalDate`, `LocalTime`, `LocalDateTime` 등)가 등장하면서 모든 것이 바뀌었다.

**최신 JPA(JPA 2.2+)와 하이버네이트는 이 `java.time` 패키지의 클래스들을 완벽하게 지원한다.** 즉, 이제는 `@Temporal` 어노테이션을 사용할 필요가 전혀 없다. 그냥 필드를 선언하기만 하면 JPA가 알아서 적절한 데이터베이스 타입으로 매핑해준다.

  * `LocalDate` (날짜) -\> `DATE` 타입
  * `LocalTime` (시간) -\> `TIME` 타입
  * `LocalDateTime` (날짜와 시간) -\> `TIMESTAMP` 타입

<!-- end list -->

```kotlin
@Entity
class Order(
    @Id @GeneratedValue
    val id: Long = 0,
    
    // LocalDate는 DB의 DATE 타입으로 매핑된다.
    val orderDate: LocalDate,

    // LocalDateTime은 DB의 TIMESTAMP 타입으로 매핑된다.
    @Column(updatable = false)
    val createdAt: LocalDateTime
)
```

위 코드처럼 `@Temporal` 없이 `LocalDate`나 `LocalDateTime` 타입을 선언하기만 하면 된다. 훨씬 코드가 간결하고 명확해졌다. `java.util.Date`를 사용하고 `@Temporal`로 매핑하는 코드는 이제 레거시 시스템에서나 볼 수 있는 '옛날 방식'이 되었다.

이제 우리는 JPA를 통해 단순 타입을 넘어 Enum과 최신 날짜/시간 타입까지 자유자재로 다룰 수 있게 되었다. 다음 절에서는 반대로 특정 필드를 데이터베이스 매핑에서 제외하거나, 아주 큰 데이터를 다루는 방법을 알아보자.