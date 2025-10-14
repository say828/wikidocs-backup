## 02\. 대안 기술 1: jOOQ (SQL 중심의 타입-세이프 빌더)

JPA의 조회(Read) 모델 한계를 극복하기 위한 가장 강력한 대안 중 하나는 단연 **jOOQ(Java Object Oriented Querying)** 다. jOOQ는 JPA와는 정반대의 철학에서 출발한 라이브러리다.

> **JPA**: 객체 모델을 먼저 만들고, 그것을 SQL로 번역한다. (객체 중심)
> **jOOQ**: 데이터베이스 스키마를 먼저 읽고, 그것을 기반으로 SQL을 작성하는 자바 코드를 생성한다. (**SQL 중심**)

jOOQ는 "SQL을 문자열로 다루는 것은 위험하고 불편하다"는 문제의식에서 출발했다. 그래서 SQL 자체를 자바 코드로, 그것도 타입-안전(Type-Safe)하게 작성할 수 있는 놀라운 방법을 제공한다.

-----

### **jOOQ의 동작 방식: 코드 생성기**

jOOQ의 핵심은 \*\*코드 생성기(Code Generator)\*\*다. jOOQ 코드 생성기를 데이터베이스 스키마에 연결하면, 다음과 같은 일들이 자동으로 일어난다.

1.  데이터베이스의 모든 테이블, 컬럼, 뷰, 스토어드 프로시저 정보를 읽어온다.
2.  각 테이블에 해당하는 자바 클래스(예: `Tables.MEMBER`)와 각 컬럼에 해당하는 `Field` 객체(예: `MEMBER.USERNAME`, `MEMBER.AGE`)를 자동으로 생성한다.

이 자동 생성된 코드를 사용하여, 우리는 마치 Querydsl처럼 유려한 Fluent API로 SQL을 작성할 수 있다.

-----

### **JPA/JPQL vs. jOOQ: 코드 비교**

'Team A'에 소속된 30세 이상 회원을 조회하는 쿼리를 비교해 보자.

**JPQL**

```jpql
SELECT m FROM Member m JOIN m.team t
WHERE t.name = 'Team A' AND m.age >= 30
```

**jOOQ**

```java
// 사전 설정으로 DSLContext 객체(create)를 주입받았다고 가정
// MEMBER와 TEAM은 jOOQ가 자동 생성한 클래스
import static com.example.db.gen.tables.Member.MEMBER;
import static com.example.db.gen.tables.Team.TEAM;

List<MemberRecord> result = create.select()
                                  .from(MEMBER)
                                  .join(TEAM).on(MEMBER.TEAM_ID.eq(TEAM.ID))
                                  .where(TEAM.NAME.eq("Team A"))
                                  .and(MEMBER.AGE.ge(30))
                                  .fetchInto(MemberRecord.class);
```

코드가 거의 실제 SQL과 일대일로 대응되는 것을 볼 수 있다. `MEMBER.AGE.ge(30)` 처럼 모든 구문이 타입-안전한 메서드 호출로 이루어지므로, 컴파일 시점에 오류를 잡을 수 있다.

jOOQ의 진정한 힘은 JPA가 힘겨워하는 복잡한 리포팅 쿼리에서 드러난다. 윈도우 함수, 인라인 뷰, 공통 테이블 표현식(CTE) 등 거의 모든 표준 SQL 및 데이터베이스 고유의 고급 기능을 자바 코드로 완벽하게 표현할 수 있다.

-----

### **JPA와 jOOQ의 현명한 공존**

그렇다면 jOOQ가 JPA보다 항상 우월할까? 그렇지 않다. 두 기술은 서로 다른 목적을 위해 태어났으며, 현명한 아키텍트는 이 둘을 함께 사용한다.

  * **쓰기(Write) 모델 & 단순 조회**: 데이터의 일관성이 중요하고, 도메인 모델의 비즈니스 로직을 처리하는 트랜잭션 영역에서는 **JPA**가 절대적으로 유리하다. 영속성 컨텍스트가 제공하는 변경 감지, 쓰기 지연, 캐시 등의 기능은 jOOQ가 따라올 수 없는 생산성을 제공한다.
  * **읽기(Read) 모델 & 복잡한 조회**: 화면에 보여주기 위한 복잡한 데이터 조회, 통계, 리포팅 등 성능과 SQL의 표현력이 중요한 영역에서는 **jOOQ**가 압도적으로 유리하다.

**CQRS 패턴**의 관점에서 본다면, **명령(Command) 측은 JPA가, 조회(Query) 측은 jOOQ가 담당**하는 것이 가장 이상적인 아키텍처 조합이라고 할 수 있다.

JPA를 깊이 있게 마스터한 개발자에게 jOOQ는 적이 아니라, JPA의 약점을 완벽하게 보완해주는 최고의 파트셔가 될 것이다. 이제 다음 절에서는 코틀린 개발자들에게 매력적인 또 다른 대안, Kotlin Exposed에 대해 알아보자.