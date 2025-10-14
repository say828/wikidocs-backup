## 03\. 주요 어노테이션(@Test, @SpringBootTest, @DataJpaTest 등) 속성 완벽 정리

이 책 전반에 걸쳐 사용된 스프링 부트와 JUnit 5의 테스트 어노테이션들은 TDD의 효율성과 표현력을 극대화하는 강력한 도구다. 이 부록은 각 어노테이션의 핵심적인 역할과 자주 사용되는 속성을 정리한 '치트 시트(Cheat Sheet)' 역할을 한다.

-----

### **JUnit 5 기본 어노테이션**

JUnit 5(Jupiter)는 모든 자바/코틀린 테스트의 기반이 되는 프레임워크다.

| 어노테이션 | 설명 |
| :--- | :--- |
| **`@Test`** | 해당 메소드가 테스트 케이스임을 나타내는 가장 기본적인 어노테이션. |
| **`@DisplayName`** | 테스트 클래스나 메소드에 공백과 특수문자를 포함한 사람이 읽기 좋은 이름을 부여한다. |
| **`@BeforeEach`** | 각각의 `@Test` 메소드가 실행되기 **전**에 매번 실행된다. 테스트 간 격리를 위한 초기화 로직에 사용. |
| **`@AfterEach`** | 각각의 `@Test` 메소드가 실행된 **후**에 매번 실행된다. 테스트가 사용한 리소스를 정리하는 데 사용. |
| **`@BeforeAll`** | 테스트 클래스의 모든 `@Test` 메소드 실행 **전** 단 한 번만 실행된다. 반드시 `companion object` 내의 `static` 메소드(`@JvmStatic`)에 적용해야 한다. 무거운 초기 설정에 사용. |
| **`@AfterAll`** | 테스트 클래스의 모든 `@Test` 메소드 실행 **후** 단 한 번만 실행된다. `@BeforeAll`과 마찬가지로 `static` 메소드에 적용. |
| **`@Nested`** | 테스트 클래스 내부에 중첩 클래스를 만들어 관련 테스트들을 논리적으로 그룹화할 수 있다. |
| **`@Disabled`** | 특정 테스트 클래스나 메소드를 일시적으로 비활성화한다. 반드시 비활성화하는 이유를 명시하는 것이 좋다. |

-----

### **Spring Boot 테스트 어노테이션**

스프링 부트는 TestContext 프레임워크를 통해 JUnit 5를 확장하여, 스프링 컨테이너와 상호작용하는 통합 테스트를 쉽게 작성할 수 있도록 지원한다.

#### **`@SpringBootTest`**

실제 애플리케이션과 거의 동일한 전체 `ApplicationContext`를 로드하는 가장 강력하고 무거운 통합 테스트 어노테이션.

| 주요 속성 | 설명 |
| :--- | :--- |
| `webEnvironment` | 웹 환경을 시뮬레이션하는 방식을 지정한다.<br>- `MOCK` (기본값): 실제 서버를 띄우지 않고, Mock 서블릿 환경을 구성. `MockMvc`를 사용한 테스트에 적합.<br>- `RANDOM_PORT`: 실제 서블릿 컨테이너(Tomcat 등)를 임의의 포트로 띄움. `TestRestTemplate`이나 `WebTestClient`로 실제 HTTP 호출 테스트 시 사용.<br>- `DEFINED_PORT`: 지정된 포트로 실제 서블릿 컨테이너를 띄움.<br>- `NONE`: 웹 환경을 전혀 구성하지 않음. |
| `classes` | `@SpringBootConfiguration`을 자동으로 찾는 대신, 테스트에 사용할 특정 설정 클래스(`@Configuration`)를 직접 지정할 수 있다. |

#### **테스트 슬라이스 (Test Slices)**

`@SpringBootTest`보다 가볍게, 특정 계층만 잘라내어 테스트할 때 사용한다.

| 어노테이션 | 설명 |
| :--- | :--- |
| **`@WebMvcTest`** | **Web 계층** 테스트용. MVC 관련 빈(`@Controller`, `@RestControllerAdvice`, `Filter` 등)만 로드한다. `@Service`나 `@Repository`는 로드하지 않으므로, `@MockBean`을 사용하여 가짜 객체로 대체해야 한다. |
| **`@DataJpaTest`** | **JPA(영속성) 계층** 테스트용. `@Repository`와 같은 JPA 관련 빈들만 로드한다. 기본적으로 인메모리 DB를 사용하도록 설정하며, 각 테스트 종료 시 트랜잭션을 롤백하여 격리를 보장한다. (이 책에서는 Testcontainers로 이 설정을 덮어썼다.) |
| **`@RestClientTest`** | `RestTemplate`이나 `WebClient`와 같은 **HTTP 클라이언트** 테스트용. `MockRestServiceServer`를 제공하여 외부 API 호출을 Mocking 할 수 있다. (이 책에서는 더 유연한 WireMock을 주로 사용했다.) |
| **`@JsonTest`** | JSON의 **직렬화/역직렬화** 로직만 독립적으로 테스트할 때 사용. `JacksonTester`나 `GsonTester` 같은 유틸리티를 제공한다. |

#### **테스트 지원 어노테이션**

| 어노테이션 | 설명 |
| :--- | :--- |
| **`@Autowired`** | 스프링 컨테이너에 등록된 빈을 테스트 코드에 주입받기 위해 사용한다. |
| **`@MockBean`** | 스프링 컨테이너에 **Mockito Mock 객체**를 빈으로 등록한다. 만약 같은 타입의 실제 빈이 있다면, 이 Mock 객체로 대체된다. `@WebMvcTest`에서 서비스 계층을 Mocking 할 때 필수적으로 사용된다. |
| **`@SpyBean`** | 스프링 컨테이너에 등록된 **실제 빈을 Mockito Spy 객체로 감싼다.** 실제 객체의 동작을 그대로 사용하면서, 일부 메소드의 행위만 Stubbing 하거나 호출 여부를 검증하고 싶을 때 사용한다. |
| **`@ActiveProfiles`** | 테스트 실행 시 활성화할 스프링 프로파일을 지정한다. (예: `@ActiveProfiles("test")`) `application-test.yml`과 같은 특정 프로파일 설정을 로드할 수 있다. |
| **`@TestPropertySource`** | `application.yml`의 속성을 테스트 중에 덮어쓰거나 새로운 속성을 추가할 수 있다. (예: `properties = ["feature.flag=true"]`) |
| **`@Transactional`** | `@DataJpaTest`에 기본적으로 포함되어 있다. 테스트 메소드에 붙이면, 해당 메소드가 끝날 때 모든 DB 변경사항을 자동으로 롤백하여 테스트 간 격리를 보장한다. |
| **`@DirtiesContext`** | 테스트가 스프링 컨테이너의 상태를 변경하여 이후 테스트에 영향을 줄 수 있을 때 사용한다. 이 어노테이션이 붙은 메소드나 클래스 실행 후, 스프링 컨테이너 캐시가 삭제되고 새로 생성된다. **테스트 속도를 매우 느리게 하므로 최후의 수단으로만 사용해야 한다.** |