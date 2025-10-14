## 01. Spring TestContext 프레임워크 심층 탐구

이 책 전반에 걸쳐 우리는 `@SpringBootTest`, `@DataJpaTest`와 같은 어노테이션을 당연하게 사용해왔다. 이러한 어노테이션 뒤에는 스프링 테스트 지원의 핵심인 **TestContext 프레임워크**가 동작하고 있다. TDD 마스터로서, 우리는 사용하는 도구의 내부 동작 원리를 깊이 있게 이해할 필요가 있다. 이는 테스트 성능을 최적화하고, 복잡한 테스트 시나리오에서 발생하는 문제들을 해결하는 데 결정적인 통찰을 제공한다.

### **TestContext 프레임워크란?**

TestContext 프레임워크는 JUnit이나 Kotest와 같은 테스트 실행 프레임워크와 스프링 컨테이너를 연결하는 교량 역할을 한다. 테스트가 실행될 때, TestContext 프레임워크는 다음과 같은 일들을 순차적으로 수행한다.

1.  테스트 클래스의 어노테이션(`@SpringBootTest`, `@ActiveProfiles` 등)을 분석하여 테스트 설정을 파악한다.
2.  설정에 맞는 스프링 `ApplicationContext`(애플리케이션 컨텍스트)를 로드하거나, 캐시된 컨텍스트를 재사용한다.
3.  테스트 클래스의 인스턴스에 의존성을 주입(`@Autowired`, `@MockBean` 등)한다.
4.  테스트 실행 생명주기의 각 시점(클래스 실행 전, 메소드 실행 전/후 등)에 적절한 콜백(`TestExecutionListener`)을 실행한다.
5.  필요한 경우 트랜잭션을 시작하고, 테스트가 끝나면 롤백한다.

### **`@SpringBootTest` vs. 테스트 슬라이스(Test Slices)**

* **`@SpringBootTest`:** '만능 칼'과 같다. 이 어노테이션은 실제 애플리케이션 실행과 거의 동일한 `@SpringBootApplication` 설정을 찾아 전체 `ApplicationContext`를 로드한다. 컨트롤러, 서비스, 리포지토리 등 모든 빈(Bean)이 스프링 컨테이너에 등록된다. API 계층 인수 테스트나 여러 계층의 통합이 중요한 테스트에 유용하지만, 모든 것을 로드하므로 테스트 실행 속도가 가장 느리다.

* **테스트 슬라이스 (`@DataJpaTest`, `@WebMvcTest` 등):** '수술용 메스'와 같다. 특정 계층(Slice)을 테스트하는 데 필요한 최소한의 빈들만 로드하여 컨텍스트를 가볍게 유지한다.
    * **`@DataJpaTest`:** JPA 관련 설정, 리포지토리, `TestEntityManager` 등 영속성 계층 테스트에 필요한 것들만 로드한다. `@Service`, `@Controller` 빈은 등록되지 않는다.
    * **`@WebMvcTest(UserController.class)`:** 스프링 MVC 관련 설정과 명시된 `UserController` 빈만 로드한다. `@Service`나 `@Repository` 빈은 등록되지 않으므로, 테스트에서는 이들을 `@MockBean`으로 대체해야 한다.

테스트 슬라이스를 사용하면 `@SpringBootTest`보다 훨씬 더 빠르고 격리된 테스트를 작성할 수 있으므로, 가능한 한 테스트의 목적에 맞는 가장 작은 슬라이스를 사용하는 것이 좋다.

### **`ApplicationContext` 캐싱: 테스트 속도의 비밀**

스프링 `ApplicationContext`를 매 테스트마다 새로 만드는 것은 매우 비싼 작업이다. TestContext 프레임워크는 이 비용을 줄이기 위해 컨텍스트를 **캐싱(caching)**한다.

**캐싱 규칙:** 테스트 스위트 내에서, **컨텍스트 설정이 동일한** 모든 테스트 클래스는 **동일한 `ApplicationContext`를 공유**한다. 여기서 '컨텍스트 설정'이란 `@SpringBootTest`의 속성, `@ActiveProfiles`, `@MockBean`의 종류 등 컨텍스트 구성에 영향을 미치는 모든 요소를 포함한다.

만약 10개의 `@DataJpaTest` 클래스가 동일한 설정을 가지고 있다면, 컨텍스트는 단 한 번만 로드되고 10개의 테스트 클래스가 이를 재사용한다. 하지만 `@ActiveProfiles("test")`가 붙은 테스트 클래스가 등장하면, TestContext 프레임워크는 이 클래스를 위한 새로운 컨텍스트를 생성하고 캐싱한다.

`@DirtiesContext` 어노테이션은 이 캐시 동작을 제어하는 데 사용된다. 특정 테스트 메소드나 클래스가 컨텍스트의 상태를 '더럽혀서' 다른 테스트에 영향을 줄 수 있을 때, 이 어노테이션을 붙이면 해당 테스트 실행 후 컨텍스트가 캐시에서 제거되고 다음 테스트를 위해 새로 만들어진다. 하지만 이는 테스트 속도를 심각하게 저하시키므로 최후의 수단으로만 사용해야 한다.

### **`TestExecutionListener`: 테스트 생명주기의 확장점**

`TestExecutionListener`는 테스트 실행의 각 단계에 개입하여 추가적인 로직을 수행할 수 있게 해주는 인터페이스다. 스프링이 제공하는 `TransactionalTestExecutionListener`(트랜잭션 관리), `DependencyInjectionTestExecutionListener`(의존성 주입) 등이 기본적으로 등록되어 동작한다. 개발자는 자신만의 Custom Listener를 만들어 특정 로직(예: 테스트 데이터 자동 생성 및 삭제)을 추가할 수도 있다.

이처럼 Spring TestContext 프레임워크의 내부 동작 원리를 이해하면, 우리는 왜 특정 어노테이션이 필요한지, 테스트가 왜 느려지는지, 그리고 복잡한 설정의 테스트를 어떻게 최적화할 수 있는지에 대한 깊은 통찰을 얻게 된다. 이는 단순히 테스트를 '작성'하는 것을 넘어, 테스트 스위트 전체를 '설계'하고 '관리'하는 TDD 전문가의 역량으로 나아가는 중요한 디딤돌이다.