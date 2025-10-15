## 03\. 대안 기술 2: Kotlin Exposed (Kotlin-native 경량 ORM)

jOOQ가 SQL의 세계를 타입-안전한 자바 코드로 가져왔다면, **Kotlin Exposed**는 코틀린 언어의 특성을 극대화하여 데이터베이스와 상호작용하는, 완전히 새로운 접근법을 제시하는 경량 ORM 프레임워크다. Exposed는 코틀린을 만든 JetBrains에서 직접 개발한 라이브러리로, '코틀린다운(Kotlin-idiomatic)' 방식으로 데이터베이스를 다루는 것이 무엇인지 보여준다. 🐘

JPA나 jOOQ와 달리, Exposed는 두 가지 스타일의 API를 동시에 제공한다.

1.  **DSL (Domain Specific Language) API**: SQL에 가까운 타입-세이프 빌더. jOOQ와 유사한 접근법이다.
2.  **DAO (Data Access Object) API**: 엔티티 객체를 통해 데이터베이스를 다루는, JPA와 유사한 접근법이다.

-----

### **Exposed의 DSL API: 코틀린으로 쓰는 SQL**

Exposed의 DSL은 코틀린의 강력한 타입 시스템과 함수형 프로그래밍 특성을 활용하여 매우 간결하고 표현력 있는 쿼리를 작성할 수 있게 해준다.

먼저, 테이블 구조를 코틀린 `object`로 직접 정의한다.

```kotlin
// 테이블 정의
object Members : Table("MEMBER") {
    val id = long("ID").autoIncrement()
    val username = varchar("USERNAME", 255)
    val age = integer("AGE")
    val teamId = long("TEAM_ID").references(Teams.id).nullable()
    override val primaryKey = PrimaryKey(id)
}
```

이 테이블 정의를 사용하여 쿼리를 작성한다.

```kotlin
// Exposed DSL 쿼리
transaction {
    val results = Members.innerJoin(Teams)
        .select { (Teams.name eq "Team A") and (Members.age greaterEq 30) }
        .orderBy(Members.username, SortOrder.DESC)
        .map { row -> MemberDto(row[Members.username], row[Teams.name]) }
}
```

`select` 블록 안에 코틀린의 중위 연산자(`eq`, `and`, `greaterEq`)를 사용하여 `WHERE` 절을 작성하는 모습이 매우 인상적이다. 마치 코틀린 코드로 SQL을 직접 짜는 듯한 직관성을 제공한다.

-----

### **Exposed의 장점과 단점**

**장점:**

1.  **완벽한 코틀린 통합**: JetBrains가 직접 만든 만큼, 코틀린 언어의 특성(DSL, 고차 함수 등)을 가장 잘 활용한다. 코틀린 개발자에게는 가장 자연스럽고 편안한 API를 제공한다.
2.  **경량성과 유연성**: JPA처럼 복잡한 영속성 컨텍스트나 프록시 개념이 없다. 더 가볍고 예측 가능하게 동작하며, DSL과 DAO API를 필요에 따라 섞어 쓸 수 있는 유연함을 제공한다.
3.  **빠른 개발 속도**: 간단한 프로젝트나 마이크로서비스에서는 JPA의 복잡한 설정 없이 빠르게 데이터 접근 코드를 작성할 수 있다.

**단점:**

1.  **JPA 기능의 부재**: 영속성 컨텍스트가 없으므로, **변경 감지(Dirty Checking), 쓰기 지연, 지연 로딩, 캐시** 등 JPA가 제공하는 수많은 성능 최적화 기능들을 사용할 수 없다. 모든 것을 개발자가 직접 관리해야 한다.
2.  **상대적으로 작은 생태계**: JPA/하이버네이트에 비해 커뮤니티의 크기나 참고 자료, 해결된 문제 사례 등이 아직은 부족한 편이다. 복잡한 문제에 부딪혔을 때 해결책을 찾기 어려울 수 있다.

-----

> **결론: Exposed는 언제 좋은 선택일까?**
>
> Kotlin Exposed는 JPA를 대체하기 위한 기술이라기보다는, **특정 목적에 더 잘 맞는 또 다른 선택지**로 이해하는 것이 좋다.
>
>   * **적합한 경우**: 간단한 CRUD 기능만 필요한 마이크로서비스, 빠르게 프로토타입을 만들어야 하는 프로젝트, 또는 JPA의 복잡성 없이 SQL을 타입-안전하게 제어하고 싶은 코틀린 순수주의자에게 매우 매력적인 선택이다.
>   * **부적합한 경우**: 복잡한 도메인 모델과 비즈니스 로직을 가진 대규모 엔터프라이즈 애플리케이션에서는 영속성 컨텍스트와 같은 JPA의 성숙한 기능들이 제공하는 안정성과 생산성을 포기하기 어렵다.
>
> -----

지금까지 우리는 JPA의 한계를 보완할 수 있는 두 가지 강력한 대안, jOOQ와 Exposed를 살펴보았다. 이 둘은 모두 JDBC 위에서 동작하는 블로킹(Blocking) I/O 기반의 기술이다. 그렇다면 애플리케이션 전체가 비동기/논블로킹으로 동작해야 하는 '리액티브(Reactive)' 세상에서는 어떤 선택지가 있을까? 다음 절에서 그 해답을 찾아보자.