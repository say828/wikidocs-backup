## 02\. 영속성 컨텍스트와 테스트: `@DataJpaTest`의 작동 원리와 트랜잭션 롤백

우리가 작성한 리포지토리 코드가 과연 예상대로 동작하는지 어떻게 신뢰할 수 있을까? `findByUsername` 쿼리 메서드가 정말 올바른 JPQL을 생성하는지, 복잡한 `Specification`이나 Querydsl 코드가 정확한 결과를 반환하는지 검증하려면 **테스트**가 필수적이다.

하지만 데이터 접근 계층을 테스트하는 것은 몇 가지 까다로운 문제를 동반한다.

1.  **테스트 환경의 독립성**: 테스트는 외부 환경(실제 DB, 네트워크 등)에 의존하지 않고 독립적으로 실행될 수 있어야 한다.
2.  **테스트 간의 격리**: 하나의 테스트가 데이터베이스의 상태를 변경했을 때, 그 변경이 다음 테스트에 영향을 주어서는 안 된다. 각 테스트는 항상 깨끗하고 예측 가능한 상태에서 시작되어야 한다.

이러한 문제를 해결하기 위해 스프링 부트는 JPA 데이터 접근 계층을 테스트하는 데 최적화된 **`@DataJpaTest`** 라는 강력한 테스트 슬라이스(Test Slice) 어노테이션을 제공한다.

-----

### **`@DataJpaTest`의 마법 같은 동작 원리**

`@DataJpaTest`를 테스트 클래스에 붙이면, 스프링 부트는 일반적인 `@SpringBootTest`와는 전혀 다른 방식으로 동작한다.

1.  **JPA 관련 빈만 로드**: `@Service`, `@Controller` 등 전체 애플리케이션 컨텍스트를 로드하는 대신, JPA 리포지토리, `EntityManager`, 데이터 소스 등 **오직 JPA와 관련된 설정과 빈들만**을 메모리에 올린다. 이를 통해 매우 가볍고 빠른 테스트 환경을 구성한다.
2.  **인메모리 데이터베이스 사용**: 기본적으로 실제 데이터베이스(MySQL, PostgreSQL 등)를 사용하는 대신, 테스트용 **H2 인메모리 데이터베이스**를 사용하도록 설정을 변경한다. 이를 통해 외부 DB 없이도 독립적인 테스트가 가능해진다. (`application.yml`에 다른 설정을 추가하여 실제 DB를 사용하도록 변경할 수도 있다.)
3.  **트랜잭션과 롤백 (가장 중요\!)**: `@DataJpaTest`는 각각의 테스트 메서드를 **하나의 트랜잭션 안에서 실행**한다. 그리고 테스트 메서드가 \*\*종료되면, 해당 트랜잭션을 즉시 롤백(Rollback)\*\*해버린다.

바로 이 '자동 롤백' 기능 덕분에 테스트 간의 완벽한 격리가 보장된다. 첫 번째 테스트에서 아무리 많은 데이터를 `save()`하고 `delete()`해도, 테스트가 끝나는 순간 모든 변경사항이 취소되므로, 두 번째 테스트는 항상 아무런 데이터도 없는 깨끗한 상태에서 시작할 수 있다.

-----

### **`@DataJpaTest`를 사용한 리포지토리 테스트**

**MemberRepositoryTest.kt**

```kotlin
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager

@DataJpaTest // JPA 테스트를 위한 슬라이스 테스트
class MemberRepositoryTest {

    @Autowired
    private lateinit var memberRepository: MemberRepository

    @Autowired
    private lateinit var testEntityManager: TestEntityManager // 테스트 전용 EntityManager

    @Test
    fun `멤버를 저장하고 username으로 조회할 수 있어야 한다`() {
        // given
        val member = Member(username = "testUser", age = 25)
        
        // when
        val savedMember = memberRepository.save(member)
        testEntityManager.flush() // 영속성 컨텍스트의 변경을 DB에 반영 (롤백될 트랜잭션 내에서)
        testEntityManager.clear() // 영속성 컨텍스트 초기화 (조회 시 1차 캐시가 아닌 DB에서 가져오도록)

        val foundMember = memberRepository.findByUsername("testUser")

        // then
        assertThat(foundMember).isNotNull
        assertThat(foundMember?.username).isEqualTo(savedMember.username)
    }

    @Test
    fun `두 번째 테스트는 첫 번째 테스트에 영향을 받지 않는다`() {
        // 이 테스트가 시작될 때, DB는 완전히 비어있다.
        val members = memberRepository.findAll()
        assertThat(members).isEmpty()
    }
}
```

  * **`@DataJpaTest`**: 이 어노테이션 하나로 모든 설정이 끝난다.
  * **`TestEntityManager`**: 실제 `EntityManager`와 거의 같지만, 테스트 환경에서 데이터를 `flush` 하거나 `clear` 하는 등 영속성 컨텍스트를 더 세밀하게 제어할 때 유용한 헬퍼다.

`@DataJpaTest`를 사용하면, 개발자는 테스트 격리를 위한 복잡한 설정이나 데이터 클린업(clean-up) 로직 없이, 오직 테스트하고 싶은 비즈니스 로직에만 집중할 수 있다.

물론, 트랜잭션이 커밋된 후의 동작을 테스트하고 싶다면 `@Transactional(propagation = Propagation.NOT_SUPPORTED)`나 `@Rollback(false)` 같은 옵션을 사용하여 자동 롤백 기능을 끌 수도 있다. 다음 절에서는 바로 이 `@Transactional` 어노테이션이 가진 더 깊은 속성들에 대해 파헤쳐 본다.