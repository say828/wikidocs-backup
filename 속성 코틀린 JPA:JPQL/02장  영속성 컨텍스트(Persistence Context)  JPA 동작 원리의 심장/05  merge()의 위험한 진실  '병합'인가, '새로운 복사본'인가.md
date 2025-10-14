## 05\. `merge()`의 위험한 진실: '병합'인가, '새로운 복사본'인가

변경 감지는 영속(Managed) 상태의 엔티티에만 동작하는 강력한 기능이다. 그렇다면 영속성 컨텍스트의 관리를 벗어난 준영속(Detached) 상태의 엔티티는 어떻게 수정해야 할까?

예를 들어, 사용자가 웹 화면에서 자신의 정보를 수정한 뒤 '저장' 버튼을 눌렀다고 상상해 보자. 컨트롤러가 받은 `User` 객체는 데이터베이스에서 조회된 적이 있지만, 이미 최초의 트랜잭션과 영속성 컨텍스트는 종료된 후다. 따라서 이 `User` 객체는 **준영속 상태**다. 이 객체의 값을 아무리 바꿔도 변경 감지는 동작하지 않는다.

이때 준영속 상태의 엔티티를 다시 영속 상태로 만들고, 그 변경 내용을 데이터베이스에 반영하기 위해 사용하는 메서드가 바로 **`entityManager.merge()`**, 우리 말로 '병합'이다. 하지만 이 '병합'이라는 이름은 개발자에게 아주 위험한 오해를 불러일으킨다.

결론부터 말하자면, **`merge()`는 준영속 상태의 엔티티를 다시 영속 상태로 만드는 것이 아니다.** `merge()`는 파라미터로 넘어온 준영속 엔티티의 정보를 바탕으로 \*\*새로운 영속 상태의 엔티티를 '반환'\*\*하는 메서드다.

-----

### **`merge()`의 정확한 동작 방식**

`val managedMember = entityManager.merge(detachedMember)` 코드가 실행될 때, 내부에서는 다음과 같은 일이 벌어진다.

1.  `merge()`는 파라미터로 넘어온 `detachedMember`의 식별자(`id`) 값을 확인한다.
2.  그 식별자 값으로 현재 영속성 컨텍스트의 1차 캐시를 조회한다.
      * **캐시에 엔티티가 있다면(Cache Hit):** 1차 캐시에 있던 영속 엔티티(`managedMember`)에 `detachedMember`의 모든 필드 값을 덮어씌운다(병합한다). 그리고 이 영속 상태의 `managedMember`를 반환한다.
      * **캐시에 엔티티가 없다면(Cache Miss):** 데이터베이스에서 해당 식별자로 엔티티를 조회한다.
          * **DB에 데이터가 있다면:** DB에서 조회한 엔티티(`dbMember`)를 1차 캐시에 올리고, `detachedMember`의 값으로 덮어씌운 후, 이 영속 상태의 `dbMember`를 반환한다.
          * **DB에 데이터가 없다면:** 새로운 엔티티를 생성하여 `detachedMember`의 값을 채운 후, 이 새로운 엔티티를 1차 캐시에 저장하고 반환한다. (SELECT 후 INSERT 실행)

**가장 중요한 진실은 이것이다: 이 모든 과정에서 파라미터로 전달된 원본 `detachedMember` 객체는 전혀 영속화되지 않고, 여전히 준영속 상태로 남아있다.**

### **코드로 증명하는 `merge()`의 함정**

이 위험한 진실을 코드로 직접 확인해 보자.

```kotlin
@Test
@Transactional
fun `merge는 새로운 영속 객체를 반환하며, 원본은 준영속으로 남는다`() {
    // 1. 초기 멤버 저장 후 준영속 상태로 만들기
    val member = Member(name = "Original Name")
    entityManager.persist(member)
    entityManager.flush()
    entityManager.clear() // member는 이제 준영속 상태

    // 2. 준영속 상태의 member 이름 변경
    member.name = "Detached Change"

    // 3. merge 실행 (나쁜 예: 반환값을 받지 않음)
    println(">>> merge() 호출 시작")
    entityManager.merge(member)
    println("<<< merge() 호출 완료")

    // 4. merge() 이후, 여전히 준영속 상태인 원본 member의 이름을 다시 변경
    member.name = "This Will Be IGNORED"
    println(">>> 원본 준영속 객체 재수정: ${member.name}")

    // 트랜잭션 커밋
}
```

`merge()`를 호출한 이후에 원본 `member` 객체의 이름을 다시 바꿔보았다. 만약 `merge()`가 `member` 객체 자체를 영속화시킨다면, 이 마지막 변경분(`"This Will Be IGNORED"`)이 변경 감지에 의해 DB에 반영되어야 한다.

결과는 어떨까? 별도의 트랜잭션으로 DB 값을 확인해 보면, 저장된 이름은 `"Detached Change"`다. `merge()` 호출 이후의 변경은 완벽하게 무시되었다.

**올바른 사용법**은 반드시 `merge()`가 반환한 새로운 영속 객체를 변수로 받아 사용하는 것이다.

```kotlin
// ... (이전 코드와 동일) ...

// 3. merge 실행 (좋은 예: 반환값을 변수로 받음)
println(">>> merge() 호출 시작")
val managedMember = entityManager.merge(member)
println("<<< merge() 호출 완료. 반환된 managedMember: ${managedMember.name}")

// 4. 이제부터는 반드시 '반환된' managedMember를 사용해야 한다.
managedMember.name = "This Will be SAVED"
println(">>> 반환된 영속 객체 수정: ${managedMember.name}")

// 트랜잭션 커밋
```

이렇게 코드를 수정하면, 데이터베이스에는 최종적으로 `"This Will be SAVED"`가 저장된다.

> **So What? (그래서 어쩌라고?)**
> `merge()`를 사용할 때는 \*\*'파라미터로 넘긴 객체는 버리고, 반환된 새 객체를 사용한다'\*\*는 규칙을 반드시 기억해야 한다. `val managedEntity = em.merge(detachedEntity);` 이 구문을 하나의 관용구처럼 외워두는 것이 좋다. 그렇지 않으면 왜 내 변경 사항이 DB에 반영되지 않는지 한참을 헤매게 되는, 아주 디버깅하기 어려운 버그의 원인이 될 수 있다.

이제 영속성 컨텍스트의 핵심 기능들을 모두 배웠다. 마지막으로, 이 영속성 컨텍스트를 개발자가 직접 제어할 수 있는 `flush()`와 `clear()` 메서드에 대해 알아보자.****