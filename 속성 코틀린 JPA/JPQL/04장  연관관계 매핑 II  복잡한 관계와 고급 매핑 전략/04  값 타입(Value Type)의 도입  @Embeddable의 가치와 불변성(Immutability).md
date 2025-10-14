## 04\. 값 타입(Value Type)의 도입: @Embeddable의 가치와 불변성(Immutability)

지금까지 우리가 다룬 모든 것은 \*\*엔티티(Entity)\*\*였다. 엔티티의 가장 큰 특징은 `@Id`로 식별되는 고유한 '정체성(Identity)'을 가지며, 생명주기가 영속성 컨텍스트에 의해 관리된다는 점이다. 하지만 우리 주변에는 정체성 없이, 그저 '값' 자체로 의미를 갖는 객체들도 많다. 주소(Address)를 생각해 보자. '서울시 강남구 테헤란로 123'이라는 주소는 그 값 자체가 중요할 뿐, 별도의 ID를 가질 필요는 없다.

이처럼 독립적인 생명주기를 갖지 않고 다른 엔티티에 종속되어, 그 값 자체로 사용되는 객체를 JPA에서는 \*\*값 타입(Value Type)\*\*이라고 부른다.

-----

### **값 타입이 없을 때의 문제점**

값 타입이라는 개념이 없다면, 우리는 엔티티에 모든 속성을 원시 타입(primitive type)으로 풀어 헤쳐서 정의해야 한다.

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long = 0,
    var username: String,

    // 주소 관련 필드들
    var city: String,
    var street: String,
    var zipcode: String
)
```

`city`, `street`, `zipcode`는 명백히 '주소'라는 하나의 개념을 이루지만, 코드 상에서는 그저 `Member`의 여러 속성 중 하나일 뿐이다. 이는 다음과 같은 문제를 낳는다.

  * **낮은 응집도**: '주소'와 관련된 로직(예: 주소 유효성 검증)이 `Member` 엔티티에 흩어지게 되어 객체의 책임이 불분명해진다.
  * **재사용성 부재**: `Company` 엔티티에도 주소가 필요하다면, `city`, `street`, `zipcode` 필드를 또다시 복사-붙여넣기 해야 한다.
  * **의미 없는 모델**: `member.getCity()`는 객체지향적이라기보다는 데이터 중심의 코드에 가깝다.

-----

### **`@Embeddable`과 `@Embedded`로 해결**

JPA는 값 타입을 만들기 위한 `@Embeddable`과, 그 값 타입을 사용하기 위한 `@Embedded`라는 두 가지 어노테이션을 제공한다.

**1. 값 타입 클래스 정의 (`@Embeddable`)**

```kotlin
@Embeddable // 나는 다른 엔티티에 포함될 수 있는 '값 타입' 클래스입니다.
data class Address(
    val city: String,
    val street: String,
    val zipcode: String
)
```

  * `@Embeddable`: 이 클래스가 값 타입임을 선언한다.
  * 기본 생성자는 필수다. (코틀린 `data class`는 자동으로 충족)

**2. 엔티티에서 값 타입 사용 (`@Embedded`)**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long = 0,
    var username: String,

    @Embedded // Address 라는 값 타입을 여기서 사용합니다.
    var address: Address
)
```

  * `@Embedded`: 값 타입 필드를 매핑한다.
  * 이렇게 매핑하면, `MEMBER` 테이블에는 `city`, `street`, `zipcode`라는 세 개의 컬럼이 이전과 동일하게 생성된다. 하지만 우리의 객체 모델은 훨씬 더 풍부하고 객체지향적으로 변했다. 이제 `member.address.city`와 같이 의미 있는 방식으로 주소 정보에 접근할 수 있다.

-----

### **값 타입의 생명주기와 불변성(Immutability)의 중요성**

값 타입은 엔티티와 달리 자신의 생명주기를 갖지 않는다. **전적으로 그것을 소유한 엔티티에 종속된다.** `Member`가 저장될 때 `address`의 필드들이 함께 저장되고, `Member`가 삭제되면 `address`도 함께 사라진다.

여기서 값 타입을 사용할 때 반드시 지켜야 할 **철칙**이 있다.

> **값 타입은 반드시 불변(Immutable) 객체로 설계해야 한다.**

만약 `Address` 클래스가 `var city: String` 처럼 변경 가능한(mutable) 필드를 가졌다고 상상해 보자.

```kotlin
val sharedAddress = Address("서울", "강남대로", "12345")
member1.address = sharedAddress
member2.address = sharedAddress // 두 엔티티가 같은 Address 인스턴스를 공유

// member1의 주소를 바꿨을 뿐인데...
member1.address.city = "부산"

// member2의 주소까지 부산으로 바뀌는 끔찍한 부작용(Side Effect) 발생!
println(member2.address.city) // "부산"
```

객체는 참조를 공유하기 때문에 이런 재앙이 발생한다. 이를 막으려면 `Address` 클래스의 모든 필드를 `val`로 선언하여 불변성을 보장해야 한다. 주소를 변경하고 싶다면, `setter`로 기존 객체의 상태를 바꾸는 것이 아니라, `copy()` 등을 사용해 **새로운 `Address` 인스턴스를 생성하여 통째로 교체**해야 한다.

`member.address = newAddress`

이 원칙만 지킨다면, 값 타입은 우리의 도메인 모델을 훨씬 더 견고하고 표현력 있게 만들어주는 강력한 무기가 될 것이다. 그렇다면 값 타입을 하나가 아닌 여러 개, 즉 컬렉션으로 다루고 싶을 때는 어떻게 해야 할까? 다음 절에서 그 방법을 알아보자.