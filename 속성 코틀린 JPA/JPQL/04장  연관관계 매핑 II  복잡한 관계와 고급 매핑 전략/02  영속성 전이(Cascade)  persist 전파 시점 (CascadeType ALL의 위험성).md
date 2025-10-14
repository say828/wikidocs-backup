## 02\. 영속성 전이(Cascade): `persist` 전파 시점 (CascadeType.ALL의 위험성)

엔티티들의 관계가 복잡해지다 보면, 하나의 로직을 처리하기 위해 여러 엔티티를 영속화해야 하는 경우가 많다. 예를 들어, 새로운 부모(Parent) 엔티티와 그에 속한 여러 자식(Child) 엔티티들을 한 번에 저장해야 한다고 생각해 보자.

```kotlin
// Cascade가 없다면...
val parent = Parent()
val child1 = Child()
val child2 = Child()

parent.addChild(child1) // 편의 메서드로 양방향 관계 설정
parent.addChild(child2)

entityManager.persist(parent)
entityManager.persist(child1) // 자식들을 일일이 영속화해야 한다.
entityManager.persist(child2)
```

`parent`를 저장하고, `child1`, `child2`도 각각 `persist` 해주어야 한다. 코드가 길어지고 번거로울 뿐만 아니라, 개발자가 실수로 자식 중 하나를 빠뜨릴 수도 있다. 마치 `parent`를 저장하면 자식들도 알아서 저장될 것처럼 느껴지는 객체지향적인 모델과 실제 코드 사이에 괴리가 발생한다.

\*\*영속성 전이(Persistence Cascade)\*\*는 바로 이 문제를 해결하기 위한 기능이다. 특정 엔티티를 영속화할 때, 그와 연관된 엔티티에도 \*\*영속성 상태를 전파(propagate)\*\*하는 것이다. `Parent`를 `persist`하면, 그에 속한 `Child`들도 자동으로 `persist` 되게 만드는 기능이다.

-----

### **`cascade` 옵션 적용하기**

`cascade` 옵션은 `@OneToMany`, `@ManyToOne` 등의 연관관계 어노테이션에 속성으로 추가할 수 있다.

**Parent.kt**

```kotlin
@Entity
class Parent(
    @Id @GeneratedValue
    val id: Long = 0,
    var name: String,

    @OneToMany(mappedBy = "parent", cascade = [CascadeType.PERSIST])
    var children: MutableList<Child> = mutableListOf()
) {
    // ... 편의 메서드
}
```

`cascade = [CascadeType.PERSIST]` 옵션을 추가했다. 이제 `Parent` 엔티티에 `persist` 작업을 수행하면, 이 작업이 `children` 컬렉션에 있는 모든 `Child` 엔티티에게도 전파된다.

```kotlin
// Cascade가 있다면!
val parent = Parent()
val child1 = Child()
val child2 = Child()

parent.addChild(child1)
parent.addChild(child2)

// parent만 영속화하면 끝!
entityManager.persist(parent)
```

코드가 훨씬 간결하고 객체지향적으로 변했다.

### **다양한 `CascadeType`**

`CascadeType`에는 `PERSIST` 외에도 여러 종류가 있다.

  * `PERSIST`: 부모를 영속화할 때 자식도 함께 영속화.
  * `MERGE`: 부모를 병합(`merge`)할 때 자식도 함께 병합.
  * `REMOVE`: 부모를 삭제할 때 자식도 함께 삭제.
  * `REFRESH`: 부모를 새로고침(`refresh`)할 때 자식도 함께 새로고침.
  * `DETACH`: 부모를 준영속 상태로 만들 때 자식도 함께 준영속.
  * `ALL`: 위의 모든 것을 한 번에 적용.

-----

### **`CascadeType.ALL`의 위험성**

`CascadeType.ALL`은 모든 영속성 작업을 전파해주므로 매우 편리해 보인다. 많은 입문서나 예제 코드에서 무분별하게 사용되는 것을 볼 수 있다. 하지만 **`CascadeType.ALL`은 매우 위험하며, 신중하게 사용해야 한다.**

문제는 `REMOVE`가 포함되어 있다는 점이다. `CascadeType.ALL`이 설정된 관계에서 부모 엔티티를 삭제하면, 그와 연관된 자식 엔티티들이 **모두 함께 데이터베이스에서 삭제**된다.

이것이 왜 위험할까? **자식 엔티티의 생명주기가 부모 엔티티에 완전히 종속적일 때만** `CascadeType.ALL`은 안전하다. 예를 들어, 게시물(`Post`)과 그에 속한 첨부파일(`Attachment`)의 관계를 생각해 보자. 게시물이 삭제되면 첨부파일도 함께 삭제되는 것이 당연하다. 이 경우에는 `CascadeType.ALL`을 사용해도 좋다.

하지만 `Team`과 `Member`의 관계는 어떨까? 팀이 해체된다고 해서 소속된 회원 정보까지 모두 DB에서 삭제되어야 할까? 그렇지 않다. 회원은 다른 팀으로 이동하거나, 탈퇴 상태로 남아있을 수 있다. 만약 이 관계에 `CascadeType.ALL`을 설정했다면, 팀을 삭제하는 순간 소속된 모든 회원의 데이터가 영원히 사라지는 끔찍한 사태가 발생할 수 있다.

> **결론: `CascadeType`은 신중하게 선택하라.**
> **자식 엔티티가 부모 엔티티와 생명주기를 완전히 같이하고, 다른 어떤 엔티티와도 연관되지 않은 '개인 소유물'일 경우에만** `CascadeType.ALL` 또는 `CascadeType.REMOVE`를 사용하라. 그 외의 경우에는 `CascadeType.PERSIST`, `CascadeType.MERGE` 등 필요한 옵션만 골라서 사용하는 것이 훨씬 안전하고 명확한 설계다.

영속성 전이는 부모의 '영속화' 상태를 전파하는 기능이다. 이와 비슷하지만 조금 다른 개념으로, 컬렉션에서 자식 엔티티가 제거되었을 때 데이터베이스에서도 자동으로 삭제되게 하는 '고아 객체'라는 기능이 있다. 다음 절에서는 이 둘의 결정적인 차이를 알아보자.