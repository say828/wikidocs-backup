# 14장: Kotlin과 JPA: 더 나은 개발 경험을 위하여

지금까지의 여정에서 우리는 코틀린을 기본 언어로 사용해왔지만, 사실 대부분의 내용은 JPA 표준과 그 구현체인 하이버네이트, 그리고 스프링 데이터 JPA에 대한 이야기였다. 즉, 자바 개발자도 거의 그대로 적용할 수 있는 내용이었다.

하지만 코틀린은 단순히 'JVM 위에서 돌아가는 또 다른 언어'가 아니다. 코틀린은 데이터 클래스(Data Class), Null-Safety, 확장 함수(Extension Function) 등 더 안전하고 간결하며 표현력 있는 코드를 작성할 수 있도록 돕는 현대적인 언어적 특성을 가지고 있다.

이번 장에서는 이 코틀린의 고유한 장점들을 JPA와 어떻게 조화롭게 엮어내어, 우리의 데이터 접근 코드를 한 단계 더 발전시킬 수 있는지 그 구체적인 기법들을 탐구한다. `merge()`의 위험성을 코틀린의 `data class`로 어떻게 피해 갈 수 있는지, 코틀린의 Null-Safety가 데이터베이스의 `NOT NULL` 제약조건과 어떻게 완벽한 시너지를 내는지, 그리고 확장 함수를 통해 반복적인 JPA 코드를 어떻게 우아하게 제거하는지 보게 될 것이다. 마지막으로, 비동기 프로그래밍의 대세인 코루틴(Coroutines)을 블로킹(Blocking) I/O 기반의 JPA와 함께 사용할 때 마주하는 한계와 그 해결책까지 논의한다.

-----

## 00\. Kotlin의 Data Class와 `copy()`: `merge()`를 대체하는 엔티티 수정 패턴

2장에서 우리는 `merge()`의 위험한 진실에 대해 배웠다. `merge()`는 파라미터로 넘어온 준영속 상태의 객체를 영속 상태로 만드는 것이 아니라, 그 객체의 모든 필드 값을 1차 캐시에 있는 영속 객체에 \*\*'묻지마 덮어쓰기'\*\*를 한 후, 새로운 영속 객체를 반환하는 복잡하고 위험한 메서드다. 웹 화면에서 일부 필드만 수정하여 전송했을 때, 전송되지 않은 필드들이 `null`로 덮어씌워져 데이터가 유실되는 끔찍한 버그의 주범이 되기도 한다.

코틀린의 `data class`와 JPA의 변경 감지(Dirty Checking)를 조합하면, 이 위험한 `merge()`를 사용하지 않고도 훨씬 더 안전하고 명시적인 '엔티티 수정 패턴'을 만들 수 있다.

-----

### **'조회 후 수정(Read-and-Update)' 패턴**

이 패턴의 핵심 아이디어는 간단하다. **"수정하고 싶은 엔티티를 영속성 컨텍스트에 먼저 올린 뒤, 그 영속 상태의 객체 값을 변경하여 변경 감지를 유발한다."**

**1. 수정 요청을 위한 DTO를 받는다.**

```kotlin
data class MemberUpdateDto(
    val username: String,
    val age: Int
)
```

**2. 서비스 계층에서 엔티티를 조회하고, DTO의 값으로 변경한다.**

```kotlin
@Service
class MemberService(
    private val memberRepository: MemberRepository
) {
    @Transactional
    fun updateMember(id: Long, updateDto: MemberUpdateDto) {
        // 1. Repository를 통해 '영속 상태'의 엔티티를 조회한다.
        val member = memberRepository.findById(id)
            .orElseThrow { EntityNotFoundException("Member not found") }

        // 2. 조회된 영속 객체의 상태를 직접 변경한다.
        //    (별도의 update 메서드를 엔티티에 만들 수도 있다.)
        member.username = updateDto.username
        member.age = updateDto.age

        // 3. @Transactional 메서드가 종료될 때, 변경 감지에 의해 UPDATE 쿼리가 자동으로 실행된다.
    }
}
```

### **왜 이 패턴이 `merge()`보다 우월한가?**

  * **명시성과 안전성**: `merge()`가 모든 필드를 깜깜이로 덮어쓰는 것과 달리, 이 패턴은 `updateDto`에 명시된 필드만 **개발자가 직접 코드로 제어하여 수정**한다. `createdAt`과 같이 수정되어서는 안 될 필드를 건드릴 위험이 원천적으로 차단된다.
  * **의도의 명확성**: 코드를 읽는 누구나 어떤 필드가 변경되는지 명확하게 알 수 있다. `merge()`처럼 내부 동작 원리를 알아야만 이해할 수 있는 코드가 아니다.

### **`data class`와 `copy()`의 역할**

이 패턴에서 코틀린의 `data class`는 그 자체로 매우 유용하다. 엔티티를 `data class`로 만들면 `equals()`, `hashCode()` 등의 메서드가 자동으로 생성되어 테스트나 비교에 편리하다.

또한, `data class`가 제공하는 **`copy()`** 메서드는 이 패턴의 정신과 맞닿아 있다. 비록 위 예제처럼 영속 상태의 객체를 직접 수정하는 것이 가장 간단하지만, 더 복잡한 시나리오에서 `copy()`는 불변성을 유지하며 객체의 일부 상태만 변경한 '새로운 버전'의 객체를 만드는 데 매우 유용하다. 이는 테스트 데이터를 만들거나, 엔티티의 현재 상태를 기반으로 DTO를 생성할 때 빛을 발한다.

> **결론: `merge()` 대신 '조회 후 수정' 패턴을 사용하라.**
> 코틀린 환경에서는 `merge()`의 위험성을 감수할 이유가 거의 없다. `@Transactional` 안에서 엔티티를 조회하고 그 상태를 직접 변경하는 것이 훨씬 더 안전하고, 명시적이며, 코틀린의 특성과도 잘 어울리는 현대적인 JPA 업데이트 방식이다.

이제 코틀린의 또 다른 강력한 무기인 Null-Safety가 JPA와 만나 어떤 시너지를 내는지 다음 절에서 알아보자.