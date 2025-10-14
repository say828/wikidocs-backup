## 04\. 완벽한 실습 환경 구축: Kotlin, Spring Boot, Gradle, JUnit 5, Kotest, MockK

이론과 철학만으로는 단 한 줄의 견고한 코드도 만들 수 없다. 우리의 TDD 여정을 성공적으로 이끌어 줄 최고의 도구들을 갖추고, 일관되고 안정적인 실습 환경을 구축하는 것은 무엇보다 중요하다. 이 책의 모든 예제 코드는 다음의 기술 스택을 기반으로 작성된다. 독자들은 이 표준 환경을 통해 어떤 예제 코드라도 마찰 없이 즉시 실행하고 따라 해 볼 수 있을 것이다.

우리가 구축할 환경은 2025년 현재, 코틀린 백엔드 개발의 가장 표준적이고 강력한 조합이다.

  * **언어 (Language):** **Kotlin (1.9.x 이상)**
      * 자바 가상 머신(JVM) 위에서 동작하며, 특유의 간결함, 표현력, 그리고 널 안정성(Null Safety)을 통해 테스트 코드와 프로덕션 코드 모두의 가독성과 안정성을 극대화한다.
  * **프레임워크 (Framework):** **Spring Boot (3.2.x 이상)**
      * Java 21과 Jakarta EE를 기반으로 하는 최신 버전으로, 의심할 여지 없는 엔터프라이즈 애플리케이션 개발의 표준이다. 우리는 스프링 부트가 제공하는 강력한 테스트 지원 기능을 적극적으로 활용할 것이다.
  * **빌드 도구 (Build Tool):** **Gradle (8.5 이상) with Kotlin DSL**
      * XML 기반의 Maven을 넘어, 코틀린 언어를 사용하여 빌드 스크립트를 작성한다. 이를 통해 타입 안정성과 스크립트의 가독성을 확보한다.
  * **핵심 테스트 라이브러리 (Core Testing Libraries):**
      * **JUnit 5:** 자바 진영의 표준 테스트 프레임워크. Spring TestContext와의 완벽한 통합을 제공한다.
      * **Kotest:** JUnit 5 위에서 동작하며, BDD(행위 주도 개발) 스타일의 서술적인 테스트 작성을 가능하게 하여 테스트를 '살아있는 문서'로 만들어주는 강력한 도구다.
      * **MockK:** 코틀린을 위해 태어난 Mocking 라이브러리. 코틀린의 언어적 특성(코루틴, 확장 함수 등)을 완벽하게 지원하여, 순수 코틀린 스타일의 깔끔한 테스트 더블(Test Double) 작성을 돕는다.
  * **개발 환경 (IDE):** **IntelliJ IDEA Ultimate**
      * 코틀린과 스프링 개발에 가장 최적화된 환경을 제공한다. Community 버전으로도 실습은 충분히 가능하다.

### **`build.gradle.kts` 설정**

프로젝트를 직접 구성하려면, 아래의 `build.gradle.kts` 파일을 시작점으로 사용하라. 이 파일에는 책의 전반부 예제를 실행하는 데 필요한 모든 의존성이 포함되어 있다.

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.2.5"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.23"
    kotlin("plugin.spring") version "1.9.23"
}

group = "com.tddmaster"
version = "0.0.1-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    
    // Kotlin
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlin:kotlin-stdlib-jdk8")

    // Core Testing (Spring Boot Test Starter가 JUnit5, AssertJ, Mockito 등을 포함)
    testImplementation("org.springframework.boot:spring-boot-starter-test")

    // Kotest for expressive tests
    testImplementation("io.kotest:kotest-runner-junit5-jvm:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core-jvm:5.8.0")

    // MockK for idiomatic Kotlin mocking
    testImplementation("io.mockk:mockk:1.13.10")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs = listOf("-Xjsr305=strict")
        jvmTarget = "21"
    }
}

tasks.withType<Test> {
    useJUnitPlatform() // JUnit 5 테스트 플랫폼 사용을 명시
}

```

위 설정을 통해 우리는 앞으로의 여정을 위한 견고하고 현대적인 기반을 마련했다. 철학적 배경을 이해했고, 로드맵을 확인했으며, 최신 도구까지 모두 갖추었다. 이제 당신은 TDD라는 강력한 기술을 통해 코드와 시스템을 완전히 통제할 준비가 되었다.

다음 장부터 우리는 이 환경 위에서 첫 번째 실패하는 테스트를 작성하며, TDD의 본질인 **Red-Green-Refactor** 사이클의 세계로 본격적으로 뛰어들 것이다.