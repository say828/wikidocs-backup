## 03\. 고아 객체(Orphan Removal)와 `CascadeType.REMOVE`의 결정적 차이

영속성 전이(`CascadeType.REMOVE`)는 부모가 삭제될 때 자식도 함께 삭제되는, 부모의 생명주기에 자식이 종속되는 기능이었다. 이와 매우 유사해 보이지만 동작의 기점이 완전히 다른 기능이 있다. 바로 \*\*고아 객체 제거(Orphan Removal)\*\*다.

'고아(Orphan)'란 말 그대로 부모와의 관계가 끊어져 버려 참조하는 곳이 없어진 엔티티를 의미한다. 고아 객체 제거는 **부모 엔티티의 컬렉션에서 자식 엔티티의 참조를 제거하면, 그 자식 엔티티를 데이터베이스에서도 자동으로 삭제**해주는 기능이다.

-----

### **`orphanRemoval = true` 옵션**

이 기능은 `@OneToMany`, `@OneToOne` 어노테이션의 `orphanRemoval` 속성을 `true`로 설정하여 활성화할 수 있다.

**Parent.kt**

```kotlin
@Entity
class Parent(
    @Id @GeneratedValue
    val id: Long = 0,
    var name: String,

    @OneToMany(mappedBy = "parent", cascade = [CascadeType.PERSIST], orphanRemoval = true)
    var children: MutableList<Child> = mutableListOf()
) {
    // ... 편의 메서드
}
```

`cascade = [CascadeType.PERSIST]` 와 함께 `orphanRemoval = true` 옵션을 추가했다. 이제 `children` 컬렉션에서 특정 `Child`를 제거하는 것만으로 `DELETE` SQL이 실행된다.

**코드로 증명하는 고아 객체 제거**

```kotlin
@Test
@Transactional
fun `고아 객체 제거 테스트`() {
    // 1. 부모와 자식 2명 저장
    val parent = Parent()
    val child1 = Child()
    val child2 = Child()
    parent.addChild(child1)
    parent.addChild(child2)
    entityManager.persist(parent)
    entityManager.flush()
    entityManager.clear()

    // 2. 부모를 다시 조회한 후, 자식 중 한 명을 컬렉션에서 제거
    val foundParent = entityManager.find(Parent::class.java, parent.id)!!
    println(">>> 자식 제거 전, 컬렉션 크기: ${foundParent.children.size}") // 2

    // 컬렉션에서 첫 번째 자식을 제거한다. 이 순간 child1은 '고아'가 된다.
    foundParent.children.removeAt(0)
    
    println(">>> 자식 제거 후, 컬렉션 크기: ${foundParent.children.size}") // 1
    println("--- 커밋 직전 ---")
} // 트랜잭션 커밋
```

위 코드를 실행하면 어떤 일이 벌어질까? 우리는 `entityManager.remove()`를 호출한 적이 전혀 없다. 그저 `List`에서 객체 하나를 제거했을 뿐이다.

**SQL 결과**

```sql
-- ... (초기 INSERT 로그) ...
>>> 자식 제거 전, 컬렉션 크기: 2
>>> 자식 제거 후, 컬렉션 크기: 1
--- 커밋 직전 ---
Hibernate: 
    /* delete com.masterclass.jpamaster.domain.Child */ delete from child where id=?
```

놀랍게도, 컬렉션에서 자식을 제거하는 행위가 `DELETE` SQL로 이어졌다. 이것이 바로 고아 객체 제거의 힘이다. 이 기능 덕분에 개발자는 컬렉션을 다루는 것만으로 데이터베이스의 상태까지 일관성 있게 관리할 수 있다.

-----

### **`orphanRemoval` vs. `CascadeType.REMOVE`: 결정적 차이**

두 기능 모두 부모와 자식의 생명주기를 연동하여 자식을 삭제한다는 공통점이 있지만, 그 \*\*삭제가 시작되는 시점(Trigger)\*\*이 완전히 다르다.

  * **`CascadeType.REMOVE`**: **부모 엔티티가 `remove()` 될 때** 동작한다. 삭제의 주체는 '부모'다. 부모가 사라지니, 자식도 따라 사라지는 것이다. 부모와의 관계가 끊어진다고 해서 자식이 삭제되지는 않는다.

  * **`orphanRemoval = true`**: **부모 엔티티의 컬렉션에서 자식의 참조가 제거될 때** 동작한다. 삭제의 주체는 '관계의 단절' 그 자체다. 자식은 다른 곳에서 참조되지 않고 오직 이 부모에게만 소속된 '고아'였기 때문에, 관계가 끊어지는 순간 존재 가치를 잃고 삭제되는 것이다. 부모 엔티티 자체는 여전히 살아있다.

**이 둘은 언제 사용해야 할까?**
`orphanRemoval = true`는 **자식 엔티티가 특정 부모 엔티티에게만 배타적으로 소유(Exclusively Owned)될 때** 사용해야 한다. 게시물(`Post`)과 첨부파일(`Attachment`)의 관계처럼, 첨부파일은 특정 게시물에만 속하며 다른 게시물과 공유되지 않는다. 이럴 때 `orphanRemoval = true`를 사용하면 첨부파일 수정 로직(기존 파일 삭제, 새 파일 추가)을 컬렉션 조작만으로 우아하게 처리할 수 있다.

두 옵션은 개념적으로 다르지만, **둘 다 활성화할 수도 있다.** 부모가 삭제될 때 자식도 삭제되고(`CascadeType.REMOVE`), 부모 컬렉션에서 제거될 때도 자식이 삭제되도록(`orphanRemoval = true`) 설정하면, 부모를 통해서만 자식의 생명주기를 완벽하게 관리할 수 있게 된다.

이제 우리는 엔티티 간의 복잡한 관계를 거의 모두 마스터했다. 마지막으로, 엔티티가 아닌 '값'의 개념을 객체지향적으로 다루는 고급 매핑 전략에 대해 다음 절에서 알아보자.