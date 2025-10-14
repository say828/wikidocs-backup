## 01\. 엔티티의 4가지 생명주기: 비영속(New), 영속(Managed), 준영속(Detached), 삭제(Removed)

영속성 컨텍스트는 엔티티를 관리하는 공간이라고 했다. 그렇다면 '관리한다'는 것은 정확히 무슨 의미일까? 엔티티는 영속성 컨텍스트와 관계를 맺는 방식에 따라 총 네 가지 상태를 가지며, 이 상태의 변화를 \*\*엔티티의 생명주기(Lifecycle)\*\*라고 부른다. 이 생명주기를 이해하는 것은 영속성 컨텍스트가 어떻게 동작하는지 파악하는 첫걸음이다.

-----

### **1. 비영속 (New / Transient)**

**비영속 상태**는 순수한 객체 상태를 의미한다. `new Member()`처럼 객체를 생성했지만, 아직 JPA의 영속성 컨텍스트에 저장하지 않은 상태다. 따라서 이 객체는 영속성 컨텍스트나 데이터베이스와는 아무런 관련이 없다. 그저 메모리에 존재하는 평범한 객체일 뿐이다.

```kotlin
// Member 객체를 생성했다.
// 이 member 객체는 현재 '비영속' 상태다.
val member = Member(id = 1L, name = "newbie") 
```

-----

### **2. 영속 (Managed)**

**영속 상태**는 엔티티가 영속성 컨텍스트에 의해 관리되는 상태를 의미한다. `EntityManager.persist()`를 호출하여 비영속 상태의 엔티티를 영속성 컨텍스트에 저장하거나, `EntityManager.find()`를 통해 데이터베이스에서 엔티티를 조회하면 해당 엔티티는 영속 상태가 된다.

영속 상태가 된 엔티티는 1차 캐시, 쓰기 지연, 변경 감지, 지연 로딩 등 JPA가 제공하는 모든 기능의 혜택을 받는다. 우리가 JPA의 마법이라고 부르는 현상들은 모두 엔티티가 바로 이 **영속 상태**에 있을 때 일어난다.

```kotlin
// 비영속 상태의 member 객체를 생성
val member = Member(id = 1L, name = "managedUser")

// EntityManager를 통해 member를 영속성 컨텍스트에 저장한다.
// 이 순간부터 member는 '영속' 상태가 된다.
entityManager.persist(member) 

// DB에서 ID가 2L인 회원을 조회한다.
// 조회된 foundMember 역시 '영속' 상태다.
val foundMember = entityManager.find(Member::class.java, 2L)
```

-----

### **3. 준영속 (Detached)**

**준영속 상태**는 엔티티가 과거에는 영속 상태였지만, 더 이상 영속성 컨텍스트가 관리하지 않는 상태를 의미한다. 영속성 컨텍스트가 제공하는 어떤 기능도 동작하지 않는다.

엔티티는 다음과 같은 경우 준영속 상태가 된다.

  * **`entityManager.detach(entity)`**: 특정 엔티티 하나만 골라 준영속 상태로 만든다.
  * **`entityManager.clear()`**: 영속성 컨텍스트를 통째로 비워 모든 엔티티를 준영속 상태로 만든다.
  * **`entityManager.close()`**: 영속성 컨텍스트를 종료하여 모든 엔티티를 준영속 상태로 만든다. (주로 트랜잭션이 끝날 때 발생)

<!-- end list -->

```kotlin
// member는 현재 영속 상태
val member = entityManager.find(Member::class.java, 1L) 

// member를 영속성 컨텍스트에서 분리한다.
// 이제 member는 '준영속' 상태가 된다.
entityManager.detach(member)

// 준영속 상태이므로, 이름을 바꿔도 UPDATE 쿼리가 실행되지 않는다.
member.name = "detachedUser" 
```

준영속 상태의 엔티티는 더 이상 JPA의 관리를 받지 않으므로, 값을 변경해도 변경 감지가 일어나지 않아 데이터베이스에 반영되지 않는다.

-----

### **4. 삭제 (Removed)**

**삭제 상태**는 엔티티를 데이터베이스에서 삭제하기로 결정한 상태다. 영속성 컨텍스트에 있는 영속 상태의 엔티티에 대해 `EntityManager.remove()`를 호출하면 해당 엔티티는 삭제 상태가 된다.

삭제 상태가 된 엔티티는 여전히 영속성 컨텍스트의 관리를 받지만, 트랜잭션이 커밋되는 시점에 데이터베이스에 `DELETE` SQL이 전달되고, 영속성 컨텍스트에서도 제거된다.

```kotlin
// member는 현재 영속 상태
val member = entityManager.find(Member::class.java, 1L)

// member를 삭제 상태로 만든다.
// 트랜잭션이 커밋될 때 실제 DELETE 쿼리가 나간다.
entityManager.remove(member)
```

이 네 가지 상태를 명확히 구분할 수 있어야 앞으로 배울 1차 캐시, 쓰기 지연, 변경 감지 등의 개념을 헷갈리지 않고 이해할 수 있다. 이제 엔티티가 영속 상태일 때 가장 먼저 누리게 되는 혜택인 **1차 캐시**에 대해 알아보자.