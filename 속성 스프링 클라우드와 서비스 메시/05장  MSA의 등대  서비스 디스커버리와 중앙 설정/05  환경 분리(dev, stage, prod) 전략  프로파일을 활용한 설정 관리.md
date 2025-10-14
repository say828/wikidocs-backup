## 환경 분리(dev, stage, prod) 전략: 프로파일을 활용한 설정 관리

지금까지 우리는 `개발(development)` 환경만을 가정하고 모든 설정을 관리했습니다. 하지만 실제 프로덕션 시스템은 **개발(dev), 스테이징(stage), 운영(prod)** 등 여러 환경으로 나뉘어 운영됩니다.

  * **개발(dev) 환경:** 개발자 PC의 H2 인메모리 DB를 사용합니다.
  * **스테이징(stage) 환경:** 운영 환경과 거의 동일하지만, 테스트용 데이터가 담긴 별도의 DB를 사용합니다.
  * **운영(prod) 환경:** 실제 고객 데이터가 담긴, 고가용성이 보장된 DB를 사용합니다.

각 환경의 DB 주소, Kafka 주소, 외부 API 키, 로그 레벨, 타임아웃 값은 모두 달라야 합니다. 이 복잡한 환경별 설정을 어떻게 효율적으로 관리할 수 있을까요?

가장 최악의 방법은 `dev`, `stage`, `prod` 환경별로 Git 브랜치를 따로 파서 설정을 관리하는 것입니다. 이는 설정 변경 시 브랜치 간의 머지(Merge) 지옥을 유발하는 안티패턴입니다.

-----

### Spring Profiles와 Spring Cloud Config의 연동

이 문제에 대한 해답은 \*\*스프링 프로파일(Spring Profiles)\*\*에 있습니다. 스프링 프로파일은 특정 프로파일(`dev`, `prod` 등)이 활성화되었을 때, 해당 프로파일에 맞는 Bean이나 설정 값을 로드하는 기능입니다.

**Spring Cloud Config**는 이 스프링 프로파일 개념을 Git 저장소의 \*\*'파일 이름 규칙'\*\*과 연동하여 매우 강력하고 직관적인 방식으로 지원합니다.

#### 설정 파일 우선순위 규칙

Config Server는 클라이언트의 `application.name`과 `active profile`에 따라, Git 저장소에서 여러 파일을 조합하여 최종 설정을 만듭니다. 파일 이름은 **더 구체적일수록 높은 우선순위**를 가집니다.

만약 `member-service`가 `prod` 프로파일로 실행된다면, Config Server는 다음 파일들을 순서대로 읽어 설정을 덮어씁니다. (아래로 갈수록 우선순위가 높음)

1.  **`application.yml`**: 모든 서비스, 모든 프로파일에 적용되는 **공통 설정** (가장 낮은 순위)
2.  **`application-prod.yml`**: `prod` 프로파일로 실행되는 **모든 서비스**에 적용되는 공통 설정
3.  **`member-service.yml`**: 프로파일과 관계없이 \*\*`member-service`\*\*에만 적용되는 설정
4.  **`member-service-prod.yml`**: `prod` 프로파일로 실행되는 \*\*`member-service`\*\*에만 적용되는 **가장 구체적인 설정** (가장 높은 순위)

-----

### 1\. 설정 Git 저장소 재구성

이 규칙에 따라, 우리의 `ecommerce-config-repo` Git 저장소 구조를 다음과 같이 재구성합니다.

**`application.yml` (글로벌 공통)**

```yaml
# Eureka 주소는 모든 환경에서 동일하다고 가정
eureka:
  client:
    service-url:
      defaultZone: http://eureka.internal:8761/eureka/
```

**`application-prod.yml` (prod 환경 공통)**

```yaml
# 운영 환경에서는 로그 레벨을 INFO로 설정
logging:
  level:
    root: INFO
```

**`member-service.yml` (member-service 공통)**

```yaml
server:
  port: 8081 # member-service의 기본 포트

member:
  greeting: "Hello, Default Member!" # 기본 인사말
```

**`member-service-dev.yml` (member-service 개발 환경)**

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:memberdb # 개발 환경에서는 H2 DB 사용
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop

member:
  greeting: "Hello, Dev Member!" # 개발 환경용 인사말
```

**`member-service-prod.yml` (member-service 운영 환경)**

```yaml
spring:
  datasource:
    # 운영 환경에서는 고가용성이 보장된 MySQL 클러스터 사용
    url: jdbc:mysql://prod-mysql.db.internal/memberdb
    username: prod_user
    password: '{cipher}...' # (암호화된 비밀번호 - 부록에서 다룸)
  jpa:
    hibernate:
      ddl-auto: validate # 운영에서는 스키마 자동 생성 금지

member:
  greeting: "Hello, Production Member!" # 운영 환경용 인사말
```

-----

### 2\. 마이크로서비스에서 프로파일 활성화

그러면 `member-service`는 자신이 `dev`인지 `prod`인지 어떻게 알 수 있을까요? 바로 **`bootstrap.yml`** 파일(또는 실행 시점의 환경 변수)을 통해 프로파일을 지정합니다.

```yaml
# member-service/src/main/resources/bootstrap.yml
spring:
  application:
    name: member-service
  cloud:
    config:
      uri: http://localhost:8888
  # 1. 이 서비스가 실행될 기본 프로파일을 지정
  profiles:
    active: dev
```

#### 프로덕션 환경에서의 프로파일 활성화 (더 중요)

실제 서버에 배포할 때는, `bootstrap.yml`의 `active` 값을 그대로 사용하지 않습니다. **실행 시점에 환경 변수나 JVM 옵션으로 프로파일을 덮어씁니다.** 이 방식이 훨씬 안전하고 유연합니다.

**JAR 실행 시 JVM 옵션으로 전달:**

```bash
java -jar -Dspring.profiles.active=prod member-service.jar
```

**Docker/Kubernetes 환경 변수로 전달:**

```dockerfile
# Dockerfile
ENV SPRING_PROFILES_ACTIVE=prod
ENTRYPOINT ["java", "-jar", "member-service.jar"]
```

이제, 동일한 `member-service.jar` 아티팩트(Artifact) 하나를 가지고, 실행 시 어떤 프로파일을 활성화하느냐에 따라 개발 DB 또는 운영 DB에 접속하는 동작을 완벽하게 분리할 수 있습니다.

-----

이것으로 05장을 마칩니다. 우리는 '서비스 디스커버리'를 통해 서비스들이 서로를 동적으로 찾게 만들었고, '중앙 설정 서버'와 '프로파일'을 통해 복잡한 분산 환경의 설정을 체계적이고 안전하게 관리하는 방법을 배웠습니다.

이제 서비스 간의 \*\*'실제 통신'\*\*은 어떻게 이루어지는지, 06장에서 본격적으로 다뤄보겠습니다.