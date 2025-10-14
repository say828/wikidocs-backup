## 04\. 변경 감지(Dirty Checking): 'setter' 호출이 'UPDATE' 쿼리로 변환되는 메커니즘

JPA를 처음 접하는 개발자들이 가장 의아해하는 지점 중 하나는 바로 "왜 `update` 메서드가 없지?"라는 것이다. `persist()`로 저장하고 `remove()`로 삭제하는데, 왜 `update()`는 없을까? 정답은 "필요 없기 때문"이다. JPA는 \*\*변경 감지(Dirty Checking)\*\*라는 지능적인 메커-니즘을 통해 객체의 변경을 자동으로 감지하고 데이터베이스에 반영한다.

이 변경 감지 기능이야말로 JPA를 단순한 SQL 매퍼가 아닌, 진정한 ORM(Object-Relational Mapping) 프레임워크로 만들어주는 핵심 기술이다. 개발자는 더 이상 SQL을 신경 쓰지 않고, 그냥 객체의 상태를 바꾸기만 하면 된다.

-----

### **변경 감지의 동작 원리**

변경 감지는 영속성 컨텍스트의 1차 캐시와 밀접한 관련이 있다. 그 동작 과정은 다음과 같다.

1.  **최초 상태 저장 (스냅샷 생성)**: 엔티티가 영속성 컨텍스트에 처음으로 들어오는 시점 (즉, `find()`로 조회되거나 `persist()` 된 직후)에, JPA는 해당 엔티티의 \*\*최초 상태를 복사하여 '스냅샷(Snapshot)'\*\*이라는 이름으로 따로 보관한다. 이 스냅샷은 1차 캐시에 엔티티와 함께 저장된다.

2.  **변경 발생**: 트랜잭션 내에서 개발자가 엔티티 객체의 값을 변경한다. (`member.name = "new name"`)

3.  **플러시(flush) 시점의 비교**: 트랜잭션이 커밋되어 영속성 컨텍스트가 플러시되는 시점이 되면, JPA는 1차 캐시에 있는 모든 엔티티에 대해 **현재 엔티티의 상태**와 **저장해 둔 스냅샷**을 하나하나 비교한다.

4.  **`UPDATE` SQL 생성 및 실행**: 만약 비교 결과, 상태가 변경된 엔티티(Dirty Entity)가 발견되면, JPA는 이 변경 사항을 반영하는 **`UPDATE` SQL을 자동으로 생성**한다. 생성된 SQL은 쓰기 지연 SQL 저장소에 등록된 후, 데이터베이스로 전송되어 실행된다.

이 모든 과정은 영속 상태의 엔티티에게만 적용된다. 준영속이나 비영속 상태의 객체는 아무리 값을 바꿔도 JPA가 그 변화를 알지 못한다.

-----

### **코드로 증명하는 변경 감지**

이제, `update()` 메서드 없이 값 변경만으로 `UPDATE` 쿼리가 생성되는 마법을 직접 확인해 보자.

```kotlin
@Test
@Transactional
fun `영속 상태의 엔티티는 변경 감지를 통해 자동으로 업데이트된다`() {
    println("--- 트랜잭션 시작 ---")
    val member = Member(name = "Original Name")
    entityManager.persist(member)
    entityManager.flush() // DB에 INSERT 쿼리 강제 실행 및 영속성 컨텍스트와 동기화
    println(">>> '${member.name}' 이름으로 멤버 저장 완료")

    println("\n>>> 멤버 이름 변경 시도")
    // 영속 상태의 member 객체의 이름을 변경한다.
    member.name = "Updated Name"
    println("<<< 멤버 이름 변경 완료. UPDATE 메서드는 호출하지 않음.")

    println("\n--- 커밋 직전 ---")
} // 이 메서드가 끝나는 시점에 @Transactional에 의해 커밋(과 플러시)이 일어난다.
```

결과는 놀랍다. 우리는 그저 코틀린 객체의 필드 값만 바꿨을 뿐이다.

```shell
--- 트랜잭션 시작 ---
Hibernate: 
    /* insert ... */ insert into member (name, id) values (?, ?)
>>> 'Original Name' 이름으로 멤버 저장 완료

>>> 멤버 이름 변경 시도
<<< 멤버 이름 변경 완료. UPDATE 메서드는 호출하지 않음.

--- 커밋 직전 ---
Hibernate: 
    /* update com.masterclass.jpamaster.domain.Member */ update member set name=? where id=?
```

로그를 보면 명확하다. `member.name = "Updated Name"` 이라는 단순한 할당문 하나가, 트랜잭션 커밋 시점에 완벽한 `UPDATE` SQL로 변환되어 실행되었다. 이것이 바로 변경 감지의 힘이다.

이 기능 덕분에 우리의 서비스 계층 코드는 훨씬 더 객체지향적으로 변할 수 있다. 더 이상 `memberDAO.update(member)` 같은 데이터 중심의 코드를 작성할 필요가 없다. 그저 비즈니스 로직에 따라 도메인 객체의 상태를 바꾸면, 데이터 동기화는 JPA가 알아서 처리해준다.

이제 JPA의 가장 강력하고 신비로운 기능 중 하나의 비밀을 알게 되었다. 하지만 준영속 상태가 된 객체를 다시 영속화하려면 어떻게 해야 할까? 바로 다음 절에서 다룰 `merge()`가 그 역할을 하지만, 여기에는 아주 위험한 함정이 숨어있다.