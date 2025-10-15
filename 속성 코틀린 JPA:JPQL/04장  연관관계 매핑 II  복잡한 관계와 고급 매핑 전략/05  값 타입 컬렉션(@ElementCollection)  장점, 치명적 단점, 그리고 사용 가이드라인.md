## 05\. 값 타입 컬렉션(@ElementCollection): 장점, 치명적 단점, 그리고 사용 가이드라인

값 타입을 하나의 필드로 사용하는 것을 넘어, 여러 개를 컬렉션으로 가지고 싶을 때도 있다. 예를 들어, 한 회원이 좋아하는 음식 여러 개(`Set<String>`)를 등록하거나, 과거에 살았던 주소 이력(`List<Address>`)을 관리하는 경우다.

이럴 때 사용하는 것이 바로 \*\*`@ElementCollection`\*\*이다. 이 어노테이션은 값 타입이나 기본 타입(String, Integer 등)의 컬렉션을 엔티티에 매핑할 수 있게 해준다. JPA는 이 컬렉션을 위해 별도의 테이블을 자동으로 생성한다.

-----

### **`@ElementCollection` 사용법**

`Member` 엔티티에 좋아하는 음식(`favoriteFoods`)과 주소 이력(`addressHistory`)을 추가해 보자.

**Member.kt**

```kotlin
@Entity
class Member(
    @Id @GeneratedValue
    val id: Long = 0,
    var username: String,

    @Embedded
    var homeAddress: Address,

    // 기본 타입 컬렉션
    @ElementCollection
    @CollectionTable(name = "FAVORITE_FOODS",
        joinColumns = [JoinColumn(name = "MEMBER_ID")])
    @Column(name = "FOOD_NAME") // 컬렉션 테이블의 값 컬럼 이름
    var favoriteFoods: MutableSet<String> = mutableSetOf(),

    // 값 타입 컬렉션
    @ElementCollection
    @CollectionTable(name = "ADDRESS_HISTORY",
        joinColumns = [JoinColumn(name = "MEMBER_ID")])
    var addressHistory: MutableList<Address> = mutableListOf()
)
```

  * `@ElementCollection`: 이 필드가 값 타입의 컬렉션임을 선언한다.
  * `@CollectionTable`: 컬렉션을 저장할 테이블을 지정한다. `name`으로 테이블 이름을, `joinColumns`로 이 테이블이 `Member`를 참조할 외래 키를 지정한다.
  * `@Column`: 기본 타입(`String`) 컬렉션의 경우, 값이 저장될 컬럼의 이름을 지정한다.

이 코드를 실행하면, `MEMBER` 테이블 외에 `FAVORITE_FOODS`와 `ADDRESS_HISTORY`라는 두 개의 테이블이 추가로 생성된다. 이 테이블들은 모두 `MEMBER_ID` 외래 키를 통해 `MEMBER` 테이블을 참조한다.

-----

### **장점과 치명적인 단점**

`@ElementCollection`은 간단한 일대다 관계를 별도의 엔티티 선언 없이 쉽게 만들 수 있다는 장점이 있다. 하지만 이 편리함 뒤에는 실무에서 사용하기 꺼려지는 **치명적인 단점**들이 존재한다.

**1. 생명주기의 완전한 종속**
값 타입 컬렉션의 요소들은 부모 엔티티에 의해 생명주기가 **완벽하게 관리**된다. `CascadeType.ALL`과 `orphanRemoval = true`가 기본으로 적용된 것과 같다. `Member`가 삭제되면 모든 주소 이력과 좋아하는 음식 데이터도 함께 삭제된다. 컬렉션에서 요소를 하나 제거하면 해당 로우가 DB에서 바로 삭제된다. 이 동작은 끌 수 없다.

**2. 식별자의 부재**
컬렉션의 요소들은 엔티티가 아닌 '값'이다. 따라서 고유한 식별자(`@Id`)가 없다. 이는 곧 다른 엔티티에서 이 값 타입 요소를 직접 참조할 수 없음을 의미한다.

**3. 가장 치명적인 문제: 비효율적인 데이터 수정**
`@ElementCollection`의 가장 큰 문제는 **컬렉션의 일부 요소만 수정하는 것이 거의 불가능**하다는 점이다.

예를 들어, `addressHistory`에 주소 3개가 있고, 그중 하나를 새로운 주소로 바꾸고 싶다고 가정해 보자. 개발자는 단순히 `List`에서 객체 하나를 교체하는 코드를 작성하겠지만, JPA는 내부적으로 다음과 같이 동작한다.

1.  해당 `MEMBER_ID`를 가진 `ADDRESS_HISTORY` 테이블의 **모든 로우를 `DELETE` 한다.**
2.  현재 `addressHistory` 컬렉션에 있는 **모든 요소를 다시 `INSERT` 한다.**

단 하나의 요소를 수정하기 위해 전체 데이터를 지우고 다시 쓰는 것이다. 컬렉션의 크기가 크다면 이는 엄청난 성능 부하를 유발한다.

-----

> **사용 가이드라인: 언제, 어떻게 사용할까?**
> `@ElementCollection`은 앞서 말한 치명적인 단점 때문에 실무에서 **매우 제한적으로, 신중하게 사용해야 한다.**
>
> **사용해도 좋은 경우:**
>
>   * `String` 같은 단순한 값들의 컬렉션일 때.
>   * 컬렉션의 크기가 매우 작고, 자주 변경되지 않을 것이 확실할 때 (예: 회원의 역할 `Set<Role>`).
>   * 데이터의 라이프사이클이 부모 엔티티와 100% 일치할 때.
>
> **대안:**
> 위의 경우가 아니라면, `@ElementCollection`을 사용하는 대신 **별도의 엔티티를 만들고 일대다(@OneToMany) 관계를 맺는 것이 훨씬 더 나은 선택**이다. `AddressHistory`라는 엔티티를 만들면, 각 주소 이력은 고유한 ID를 갖게 되고, 수정이나 삭제도 훨씬 더 효율적으로 처리할 수 있다.
>
> -----

이것으로 복잡한 관계와 값 타입 매핑에 대한 여정을 마친다. 우리는 일대일, 다대다 관계의 미묘한 지점들을 파고들었고, 영속성 전이, 고아 객체, 그리고 값 타입이라는 강력하지만 주의가 필요한 고급 기능들을 배웠다. 이제 객체지향 프로그래밍의 핵심 특징 중 하나인 '상속'을 관계형 데이터베이스에서 어떻게 구현하는지, 다음 5장에서 그 흥미로운 전략들을 살펴보자.