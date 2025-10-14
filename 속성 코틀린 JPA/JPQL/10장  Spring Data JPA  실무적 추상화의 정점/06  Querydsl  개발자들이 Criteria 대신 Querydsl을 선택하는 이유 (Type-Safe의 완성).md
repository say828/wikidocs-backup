## 06\. Querydsl: 개발자들이 Criteria 대신 Querydsl을 선택하는 이유 (Type-Safe의 완성)

`Specification`은 Criteria API를 실용적으로 만들어주었지만, 여전히 코드의 복잡성과 문자열 기반의 필드 접근이라는 한계를 완전히 벗어나지는 못했다. 그렇다면 타입-안전(Type-Safe)하면서도, JPQL처럼 직관적이고 읽기 쉬우며, 동적 쿼리까지 완벽하게 지원하는 '꿈의 기술'은 없을까?

그 질문에 대한 Java/Kotlin 진영의 대답이 바로 **Querydsl**이다. Querydsl은 JPA 표준은 아니지만, 실무에서 표준 기술처럼 널리 사용되는 강력한 오픈소스 쿼리 빌더 라이브러리다.

-----

### **Querydsl이란 무엇인가?**

Querydsl은 코틀린/자바 코드를 사용하여 SQL/JPQL 쿼리를 생성할 수 있게 해주는 프레임워크다. Criteria API와 목적은 같지만, 그 접근 방식과 결과물의 차이는 하늘과 땅 차이다. Criteria가 장황한 메서드 호출의 연속이라면, Querydsl은 마치 SQL을 직접 작성하는 것처럼 자연스럽고 유려한 \*\*'Fluent API'\*\*를 제공한다.

**핵심 원리: Q 클래스**
Querydsl은 프로젝트를 컴파일할 때, Annotation Processor를 통해 기존에 작성된 엔티티 클래스(`Member`)를 분석하여, 그와 똑같은 메타데이터를 가진 \*\*Q 클래스(`QMember`)\*\*를 자동으로 생성한다.

`QMember` 클래스 안에는 `Member` 엔티티의 각 필드에 해당하는 `Path` 객체들(`QMember.member.username`, `QMember.member.age`)이 정적으로 정의되어 있다. 우리는 이 Q 클래스를 사용하여 쿼리를 작성하게 되며, 이 과정에서 모든 것이 타입-안전하게 검증된다.

-----

### **Criteria vs. Querydsl: 코드의 극명한 차이**

'팀 A'에 소속된 30세 이상 회원을 이름 내림차순으로 조회하는 쿼리를 다시 한번 비교해 보자.

**Criteria (복잡하고 난해함)**

```kotlin
val cb = entityManager.criteriaBuilder
val cq = cb.createQuery(Member::class.java)
val m = cq.from(Member::class.java)
val t = m.join<Member, Team>("team")
// ... (이하 생략) ...
```

**Querydsl (직관적이고 간결함)**

```kotlin
// 사전 설정으로 JPAQueryFactory 빈을 주입받았다고 가정
val member = QMember.member
val team = QTeam.team

val result = queryFactory
    .selectFrom(member)
    .join(member.team, team)
    .where(
        team.name.eq("Team A"),
        member.age.goe(30) // goe: Greater Or Equal (>=)
    )
    .orderBy(member.username.desc())
    .fetch()
```

차이가 느껴지는가? Querydsl 코드는 마치 SQL을 읽는 것처럼 자연스럽다. `selectFrom`, `join`, `where`, `orderBy` 등 모든 것이 명확하며, `team.name.eq("Team A")` 처럼 모든 조건이 타입-안전한 메서드 호출로 이루어진다. 오타나 타입 불일치는 컴파일 시점에 모두 걸러진다.

동적 쿼리는 더욱 강력하다. `BooleanBuilder`나 `where` 절의 다중 파라미터를 사용하면, `Specification`보다 훨씬 더 간결하고 우아하게 동적 조건을 조합할 수 있다.

-----

> **결론: 실무에서는 Querydsl이 정답이다.**
> Criteria API와 `Specification`은 JPA의 표준 기술이라는 상징적인 의미는 있지만, 생산성, 가독성, 유지보수성 모든 측면에서 Querydsl의 상대가 되지 못한다.
>
>   * **JPQL**: 정적 쿼리나 간단한 DTO 조회에 적합하다.
>   * **Querydsl**: 복잡한 조회, 동적 쿼리, 타입-안전성이 중요한 모든 곳에 적합하다.
>
> 현대적인 스프링 기반의 실무 프로젝트에서 복잡한 조회 기능을 구현해야 한다면, Querydsl을 도입하는 것은 선택이 아닌 필수다.

-----

이것으로 JPA를 기반으로 한 데이터 접근 기술의 정점, 스프링 데이터 JPA와 Querydsl까지 모두 정복했다. 우리는 순수 JPA의 동작 원리부터 시작하여, 스프링이 제공하는 놀라운 생산성의 세계까지 모두 경험했다.

이제 우리는 데이터 모델링과 조회에 대한 거의 모든 무기를 갖추었다. 하지만 진짜 실력은 위기 상황에서 드러나는 법. 실제 운영 환경에서 마주하게 될 가장 치명적인 문제, 바로 **성능 저하**를 어떻게 진단하고 해결할 것인가? 다음 11장부터는 이 책의 하이라이트인 **성능 최적화**의 세계로 깊이 들어가 볼 것이다. 가장 먼저, N+1 문제를 다시 한번 소환하여 그 근본적인 해결책들을 탐구한다.