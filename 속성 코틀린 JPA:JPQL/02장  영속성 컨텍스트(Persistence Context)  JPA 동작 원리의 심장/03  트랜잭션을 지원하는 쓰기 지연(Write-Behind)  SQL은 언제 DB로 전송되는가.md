## 03\. 트랜잭션을 지원하는 쓰기 지연(Write-Behind): SQL은 언제 DB로 전송되는가

1차 캐시가 조회(SELECT) 성능을 최적화하는 기술이라면, \*\*쓰기 지연(Write-Behind)\*\*은 등록/수정/삭제(INSERT/UPDATE/DELETE) 성능을 최적화하는 매우 중요한 기술이다.

`entityManager.persist(member)`를 호출했을 때, 많은 개발자들은 즉시 데이터베이스에 `INSERT` 쿼리가 전송될 것이라고 예상한다. 하지만 `IDENTITY` 키 생성 전략을 사용하지 않는 한, JPA는 그렇게 섣불리 행동하지 않는다.

대신, JPA는 다음과 같은 방식으로 동작한다.

1.  `persist()`가 호출되면, 해당 엔티티는 1차 캐시에 저장되는 동시에, 이 엔티티를 데이터베이스에 저장하기 위한 **`INSERT` SQL 쿼리가 생성**된다.
2.  생성된 SQL은 즉시 DB로 전송되는 것이 아니라, 영속성 컨텍스트 내부의 **'쓰기 지연 SQL 저장소(ActionQueue)'** 라는 곳에 차곡차곡 쌓인다.
3.  `UPDATE`나 `DELETE` 쿼리 역시 같은 방식으로 이 저장소에 쌓인다.
4.  SQL들은 계속 저장소에 대기하고 있다가, 트랜잭션을 \*\*커밋(commit)\*\*하는 시점에 쌓여있던 모든 SQL들이 **한꺼번에 데이터베이스로 전송**된다. 이 과정을 \*\*플러시(flush)\*\*라고 부른다.

이 쓰기 지연 기능 덕분에 우리는 상당한 성능 이점을 얻을 수 있다. 짧은 트랜잭션 안에서 100개의 데이터를 저장한다고 가정해 보자. 만약 `persist()`를 호출할 때마다 DB와 통신한다면, 네트워크를 100번 왕복해야 한다. 하지만 쓰기 지연을 활용하면, 100개의 `INSERT` 쿼리를 모았다가 단 한 번의 네트워크 통신으로 모두 처리할 수 있다. JDBC가 제공하는 **배치(batch) 기능**을 활용하면 이 효과를 극대화할 수 있다.

### **코드로 증명하는 쓰기 지연**

이번에도 코드를 통해 쓰기 지연이 실제로 어떻게 동작하는지 확인해 보자.

```kotlin
@Test
@Transactional
fun `쓰기 지연 SQL 저장소는 쿼리를 모았다가 한번에 전송한다`() {
    println("--- 트랜잭션 시작 ---")

    val memberA = Member(name = "Member A")
    val memberB = Member(name = "Member B")

    println(">>> memberA 영속화 시도")
    entityManager.persist(memberA)
    println("<<< memberA 영속화 완료. SQL은 아직 전송되지 않음.")

    println("\n>>> memberB 영속화 시도")
    entityManager.persist(memberB)
    println("<<< memberB 영속화 완료. SQL은 아직 전송되지 않음.")

    println("\n--- 커밋 직전 ---")
} // 이 메서드가 끝나는 시점에 @Transactional에 의해 커밋이 일어난다.
```

`@Id`의 생성 전략이 `SEQUENCE`나 `AUTO`로 되어있다고 가정하고 (또는 `IDENTITY`가 아니라고 가정하고) 위 코드를 실행하면, 그 결과는 매우 흥미롭다.

```shell
--- 트랜잭션 시작 ---
>>> memberA 영속화 시도
<<< memberA 영속화 완료. SQL은 아직 전송되지 않음.

>>> memberB 영속화 시도
<<< memberB 영속화 완료. SQL은 아직 전송되지 않음.

--- 커밋 직전 ---
Hibernate: 
    /* insert for com.masterclass.jpamaster.domain.Member */ insert into member (name, id) values (?, ?)
Hibernate: 
    /* insert for com.masterclass.jpamaster.domain.Member */ insert into member (name, id) values (?, ?)
```

예상대로다\! `persist()`가 호출되는 시점에는 아무런 `INSERT` 로그가 보이지 않는다. 대신, 테스트 메서드가 모두 실행되고 **트랜잭션이 커밋되는 마지막 순간**에 두 개의 `INSERT` SQL이 한꺼번에 실행되는 것을 명확하게 확인할 수 있다.

이것이 바로 JPA가 애플리케이션과 데이터베이스 사이에서 똑똑한 버퍼(buffer) 역할을 하며 성능을 최적화하는 방식이다.

그런데 여기서 한 가지 궁금증이 생긴다. `INSERT`와 `DELETE`는 `persist()`, `remove()`라는 명시적인 메서드가 있으니 이해가 된다. 그렇다면 `UPDATE`는? 우리는 엔티티를 수정하기 위해 `update()` 같은 메서드를 호출한 적이 없다. 그런데 어떻게 JPA는 엔티티의 변경을 감지하고 `UPDATE` SQL을 만들어 '쓰기 지연 SQL 저장소'에 담는 걸까? 이 놀라운 메커니즘의 비밀이 바로 다음 절의 주제, \*\*변경 감지(Dirty Checking)\*\*다.