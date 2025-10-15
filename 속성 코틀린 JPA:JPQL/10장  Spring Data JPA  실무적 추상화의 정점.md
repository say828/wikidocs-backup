# 10장: Spring Data JPA: 실무적 추상화의 정점

지금까지 우리는 JPA의 표준 API를 사용하여 데이터베이스와 상호작용하는 저수준(low-level)의 기술들을 깊이 있게 탐구했다. `EntityManager`를 직접 다루고, JPQL 문자열을 한 땀 한 땀 작성했으며, Criteria와 네이티브 SQL의 복잡성까지 마주했다. 이 모든 것은 JPA의 동작 원리를 이해하고 어떤 문제든 해결할 수 있는 튼튼한 기본기를 다지기 위한 필수적인 과정이었다.

하지만 솔직히 말해보자. 매번 똑같은 `em.persist(entity)`, `em.find(...)`, `em.createQuery(...)` 코드를 반복해서 작성하는 것은 지루하고 생산성이 떨어지는 일이다. 이런 반복적인 데이터 접근 로직을 구현하는 계층을 우리는 보통 DAO(Data Access Object) 또는 리포지토리(Repository)라고 부른다.

이번 장에서는 이 지루한 반복 작업을 마법처럼 사라지게 만드는, 실무 개발의 최종병기 \*\*스프링 데이터 JPA(Spring Data JPA)\*\*의 세계로 떠난다. 스프링 데이터 JPA는 JPA를 한 단계 더 감싸서, 개발자가 거의 아무런 코드도 작성하지 않고 데이터 접근 계층을 완성할 수 있게 해주는 놀라운 추상화 라이브러리다.

우리는 스프링 데이터 JPA가 어떤 철학으로 반복적인 코드를 제거하는지부터 시작하여, 그 핵심인 `JpaRepository` 인터페이스를 심층 분석할 것이다. 또한, 메서드 이름만으로 쿼리를 자동 생성하는 '쿼리 메서드'의 경이로운 기능과, 복잡한 동적 쿼리를 우아하게 처리하는 '명세(Specification)'와 'Querydsl'의 통합까지, 스프링 데이터 JPA가 제공하는 생산성의 정점을 맛보게 될 것이다. 이 장을 마치면, 여러분은 JPA라는 강력한 엔진 위에 스프링이라는 잘 만든 자동 변속기를 얹어, 최고의 속도와 효율로 개발을 즐길 수 있게 될 것이다.

-----

## 00\. Spring Data JPA의 철학: 반복적인 DAO 코드의 완전한 제거

스프링 데이터 JPA가 등장하기 전, JPA를 사용한 전형적인 리포지토리 클래스는 다음과 같은 모습이었다.

**과거의 방식: 순수 JPA를 사용한 리포지토리 구현**

```kotlin
@Repository // 스프링 빈으로 등록
class MemberRepositoryOld {

    @PersistenceContext // EntityManager 주입
    private lateinit var em: EntityManager

    fun save(member: Member): Member {
        em.persist(member)
        return member
    }

    fun findById(id: Long): Member? {
        return em.find(Member::class.java, id)
    }

    fun findAll(): List<Member> {
        return em.createQuery("SELECT m FROM Member m", Member::class.java)
            .resultList
    }

    fun delete(member: Member) {
        em.remove(member)
    }
    //... 기타 여러 CRUD 메서드들
}
```

`save`, `findById`, `findAll`... 모든 엔티티에 대해 이와 비슷한 코드를 반복해서 작성해야 했다. 엔티티의 종류가 100개라면, 100개의 리포지토리 클래스와 수많은 중복 코드가 생겨난다.

스프링 데이터 JPA는 바로 이 지점에서 질문을 던진다. **"어차피 다 똑같은 코드인데, 이걸 개발자가 직접 작성해야만 할까?"**

이 질문에 대한 해답이 바로 스프링 데이터 JPA의 핵심 철학, \*\*'반복적인 데이터 접근 코드의 완전한 제거'\*\*다. 스프링 데이터 JPA는 몇 가지 간단한 규칙(Convention)만 따르면, 이 모든 지루한 코드를 **런타임에 자동으로 생성**하여 개발자에게 제공한다.

-----

### **마법의 비밀: 인터페이스와 프록시**

스프링 데이터 JPA가 마법을 부리는 방법은 다음과 같다.

1.  개발자는 구현 클래스 대신 \*\*인터페이스(Interface)\*\*만 정의한다. 그리고 스프링 데이터 JPA가 제공하는 `JpaRepository` 인터페이스를 상속받는다.

**현대의 방식: 스프링 데이터 JPA를 사용한 리포지토리 정의**

```kotlin
// 끝! 구현 클래스는 필요 없다.
interface MemberRepository : JpaRepository<Member, Long>
```

2.  애플리케이션이 시작될 때, 스프링 데이터 JPA는 `JpaRepository`를 상속받은 모든 인터페이스를 찾아낸다.
3.  그리고 해당 인터페이스에 대한 **구현체를 프록시(Proxy) 기술을 사용하여 동적으로 생성**한 뒤, 스프링 빈으로 등록한다. 이 구현체 안에는 `save()`, `findById()`, `findAll()`과 같은 모든 기본 CRUD 메서드가 이미 만들어져 있다.
4.  개발자는 서비스 클래스에서 이 `MemberRepository` 인터페이스를 주입받아, 마치 구현체가 원래부터 존재했던 것처럼 자유롭게 사용하면 된다.

<!-- end list -->

```kotlin
@Service
class MemberService(
    private val memberRepository: MemberRepository // 인터페이스를 주입받는다!
) {
    @Transactional
    fun join(member: Member) {
        memberRepository.save(member) // 그냥 호출하면 된다.
    }

    fun findMember(id: Long): Member? {
        return memberRepository.findById(id).orElse(null)
    }
}
```

단 한 줄의 구현 코드 없이, 인터페이스 정의만으로 완벽하게 동작하는 데이터 접근 계층이 완성되었다. 이것이 스프링 데이터 JPA가 제공하는 혁신적인 패러다임이다. 개발자는 더 이상 CRUD의 구현이라는 반복 작업에 시간을 낭비하지 않고, 오직 핵심적인 비즈니스 로직에만 집중할 수 있게 된다.

이제, 우리가 상속받은 `JpaRepository` 인터페이스 안에는 어떤 기능들이 숨어있는지 다음 절에서 자세히 살펴보자.