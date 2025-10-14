## 04\. 이 책을 위한 학습 환경 준비: Kotlin/Java, H2 DB, IntelliJ, Gradle/Maven

백문이 불여일견(百聞不如一見), 백견이 불여일타(百見不如一打)다. 이론을 배우는 가장 좋은 방법은 직접 코드를 작성하고 실행하며 그 결과를 눈으로 확인하는 것이다. 이제부터 이 책의 모든 예제를 막힘없이 실행할 수 있는 표준 개발 환경을 구축해 보자.

**[학습 환경 요구사항]**

1.  **IDE**: **IntelliJ IDEA** Ultimate 또는 Community Edition. (스프링 부트 지원이 강력한 Ultimate 버전을 권장한다.)
2.  **JDK**: **Java 17 이상**. (스프링 부트 3.x 버전의 최소 요구사항은 Java 17이다.)
3.  **언어**: **Kotlin 1.9+**
4.  **빌드 도구**: **Gradle 8.x+** 또는 **Maven 3.8+**
5.  **데이터베이스**: **H2 In-Memory Database** (설치가 필요 없으며, 애플리케이션 실행 시 메모리에서 동작하여 학습용으로 아주 편리하다.)

스프링 부트 프로젝트는 [Spring Initializr](https://start.spring.io/)를 통해 쉽게 생성할 수 있다. 아래 의존성을 포함하여 프로젝트를 생성하거나, 기존 프로젝트에 아래 내용을 추가하자.

-----

### **Gradle 설정 (`build.gradle.kts`)**

코틀린 진영에서 많이 사용하는 Gradle 기반의 설정이다. `plugins` 블록에 코틀린과 JPA를 위한 플러그인을 반드시 추가해야 한다.

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.2.5"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.23"
    kotlin("plugin.spring") version "1.9.23"
    kotlin("plugin.jpa") version "1.9.23" // JPA를 위한 no-arg 플러그인
}

group = "com.masterclass"
version = "0.0.1-SNAPSHOT"

java.sourceCompatibility = JavaVersion.VERSION_17

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

// JPA 엔티티는 기본 생성자가 필요하므로 no-arg 플러그인을 설정한다.
noArg {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.Embeddable")
    annotation("jakarta.persistence.MappedSuperclass")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"
        jvmTarget = "17"
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

> **[잠깐\!] 코틀린과 JPA 플러그인**
>
>   * `kotlin("plugin.jpa")`: JPA 명세상 엔티티 클래스는 \*\*인자가 없는 기본 생성자(No-Argument Constructor)\*\*를 반드시 가져야 한다. 하지만 코틀린의 `data class`는 주 생성자의 파라미터에 따라 생성자가 만들어지므로 이 규칙을 위반한다. `no-arg` 플러그인은 `@Entity`가 붙은 클래스에 자동으로 기본 생성자를 추가해주는 마법 같은 역할을 한다.
>   * `kotlin("plugin.spring")`: 스프링의 클래스와 메서드는 기본적으로 `final`이라 상속이나 오버라이딩이 불가능하다. 스프링 AOP 등은 프록시 객체를 만들기 위해 클래스를 상속해야 하는 경우가 많다. 이 플러그인은 스프링 관련 어노테이션이 붙은 클래스를 자동으로 `open` 시켜준다.

-----

### **애플리케이션 설정 (`src/main/resources/application.yml`)**

데이터베이스 연결 정보와 JPA 동작에 필요한 핵심 옵션들을 설정한다.

```yaml
spring:
  # H2 데이터베이스 설정
  datasource:
    url: jdbc:h2:mem:testdb # 메모리 모드로 동작하는 H2 DB
    username: sa
    password:
    driver-class-name: org.h2.Driver

  # JPA 및 하이버네이트 설정
  jpa:
    hibernate:
      # create: 기존 테이블 삭제 후 다시 생성 (개발용)
      # none: 운영 환경에서 사용
      ddl-auto: create
    properties:
      hibernate:
        # 콘솔에 출력되는 SQL을 보기 좋게 포맷팅
        format_sql: true
        # 주석으로 어떤 JPQL에서 생성된 SQL인지 표시
        use_sql_comments: true
    # 콘솔에 하이버네이트가 실행하는 SQL을 출력
    show-sql: true

# 로깅 레벨 설정
logging:
  level:
    # 하이버네이트가 SQL 바인딩 파라미터를 로그로 남기도록 설정
    org.hibernate.type.descriptor.sql: trace
```

위 `logging` 설정은 `show-sql`보다 더 강력하다. `show-sql`은 단순히 실행된 SQL 구문만 보여주지만, 로깅 레벨을 `trace`로 설정하면 SQL의 `?` 위치에 어떤 파라미터가 바인딩되었는지 까지 모두 보여주므로 디버깅에 훨씬 유용하다.

이것으로 모든 준비가 끝났다. 이제 우리는 JPA와 하이버네이트의 강력한 기능을 탐험할 준비가 된 셈이다. 다음 절에서는 드디어 첫 번째 엔티티를 만들고 데이터베이스에 저장하는 'Hello, Entity' 예제를 실행해 보자.