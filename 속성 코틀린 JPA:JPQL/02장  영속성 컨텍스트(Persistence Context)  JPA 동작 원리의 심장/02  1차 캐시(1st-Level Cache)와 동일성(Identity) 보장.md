## 02\. 1차 캐시(1st-Level Cache)와 동일성(Identity) 보장

엔티티가 영속 상태가 되면 영속성 컨텍스트 내부의 특별한 공간에 저장된다고 했다. 이 공간의 정체가 바로 \*\*1차 캐시(1st-Level Cache)\*\*다. 영속성 컨텍스트는 내부에 이 1차 캐시를 가지고 있으며, 영속 상태의 모든 엔티티는 이곳에 저장된다.

이 1차 캐시는 `Map`과 유사한 구조를 가진다. **키(Key)는 엔티티의 `@Id` 값**이고, **값(Value)은 엔티티 객체 인스턴스 그 자체**다.

JPA가 1차 캐시를 활용하는 방식은 다음과 같다.

1.  **`entityManager.persist(member)` 호출**: `member` 엔티티가 1차 캐시에 저장된다. 이때 키는 `member`의 `id`가 된다. (IDENTITY 전략이 아닐 경우, 이 시점까지는 DB에 `INSERT` SQL을 보내지 않는다.)

2.  **`entityManager.find(Member.class, memberId)` 최초 호출**:

      * 먼저 1차 캐시에서 `memberId`를 키로 가진 엔티티가 있는지 찾아본다.
      * **(Cache Miss)** 만약 없다면, 데이터베이스로 `SELECT` SQL을 보낸다.
      * 조회 결과를 바탕으로 엔티티 객체를 생성하여 **1차 캐시에 저장**한다.
      * 생성된 엔티티 객체를 반환한다.

3.  **`entityManager.find(Member.class, memberId)` 재호출**:

      * 1차 캐시에서 `memberId`를 키로 가진 엔티티를 찾아본다.
      * **(Cache Hit)** 이번에는 캐시에 엔티티가 존재하므로, DB를 조회하지 않고 **캐시에 있는 엔티티 객체를 즉시 반환**한다.

이것이 1차 캐시의 핵심 동작 원리다. 같은 트랜잭션 안에서 동일한 엔티티를 반복적으로 조회할 경우, 두 번째 조회부터는 데이터베이스를 전혀 거치지 않으므로 성능상 큰 이점을 얻을 수 있다.

### **코드로 증명하는 1차 캐시**

말로만 듣는 것보다 직접 눈으로 확인해 보자.

```kotlin
@Test
@Transactional
fun `1차 캐시는 반복 조회를 효율적으로 처리한다`() {
    // 테스트용 멤버를 미리 저장
    val member = Member(name = "Master Kim")
    entityManager.persist(member)
    entityManager.flush() // DB에 INSERT 쿼리 강제 실행
    entityManager.clear() // 영속성 컨텍스트 초기화
    println("--- 초기 데이터 저장 완료 ---")

    // 1. 첫 번째 조회
    println("\n>>> 1. 첫 번째 조회 시작")
    val foundMember1 = entityManager.find(Member::class.java, member.id)
    println("<<< 1. 첫 번째 조회 완료: ${foundMember1.name}")

    // 2. 동일한 엔티티를 다시 조회
    println("\n>>> 2. 두 번째 조회 시작")
    val foundMember2 = entityManager.find(Member::class.java, member.id)
    println("<<< 2. 두 번째 조회 완료: ${foundMember2.name}")
}
```

위 테스트를 실행하면 다음과 같은 로그를 볼 수 있다.

```shell
--- 초기 데이터 저장 완료 ---

>>> 1. 첫 번째 조회 시작
Hibernate: 
    /* select ... from member m1_0 where m1_0.id=? */ select ...
<<< 1. 첫 번째 조회 완료: Master Kim

>>> 2. 두 번째 조회 시작
<<< 2. 두 번째 조회 완료: Master Kim
```

결과가 보이는가? 첫 번째 조회 시점에는 `SELECT` SQL이 분명히 실행되었지만, **두 번째 조회 시점에는 아무런 SQL도 실행되지 않았다.** JPA가 `member.id`를 키로 가진 `Member` 엔티티가 이미 1차 캐시에 있음을 확인하고, DB까지 가지 않고 캐시에서 객체를 바로 꺼내주었기 때문이다.

-----

### **영속 엔티티의 동일성(Identity) 보장**

1차 캐시는 중요한 특징을 하나 더 파생시킨다. 바로 **같은 트랜잭션 내에서 영속 엔티티의 동일성을 보장**한다는 점이다. '동일성'은 자바/코틀린의 `==` 비교 연산자가 `true`를 반환하는 것, 즉 두 변수가 메모리의 **완전히 같은 인스턴스**를 참조하고 있음을 의미한다.

1차 캐시 덕분에, 같은 ID로 여러 번 조회하더라도 JPA는 항상 최초에 캐싱된 동일한 객체 인스턴스를 반환한다.

```kotlin
// ... 이전 테스트 코드에 이어서 ...
println("\n>>> 동일성 비교 시작")
println("두 객체는 동일한가? --> ${foundMember1 === foundMember2}") // === 는 코틀린의 참조 비교
```

결과는 당연히 다음과 같다.

```shell
>>> 동일성 비교 시작
두 객체는 동일한가? --> true
```

이 동일성 보장 기능 덕분에 개발자는 마치 자바 컬렉션에서 객체를 다루는 것처럼 일관된 프로그래밍 모델을 유지할 수 있다. 만약 조회할 때마다 다른 객체 인스턴스가 반환된다면, 애플리케이션 로직은 훨씬 더 복잡해졌을 것이다.

1차 캐시는 이처럼 조회 성능을 최적화하고 객체의 일관성을 유지하는 중요한 역할을 한다. 그렇다면 쓰기 작업은 어떨까? 다음 절에서는 JPA가 `INSERT` SQL을 바로 보내지 않고 아껴두었다가 한 번에 처리하는 '쓰기 지연'의 비밀을 파헤쳐 본다.