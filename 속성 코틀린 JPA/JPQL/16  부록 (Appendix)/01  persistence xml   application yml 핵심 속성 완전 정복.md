## 00\. persistence.xml / application.yml 핵심 속성 완전 정복

JPA의 모든 동작 방식과 성능은 결국 설정 파일 하나에서 시작된다. 전통적인 JPA 환경에서는 `META-INF/persistence.xml` 파일이 그 역할을 했지만, 스프링 부트 환경에서는 **`application.yml`** (또는 `.properties`) 파일이 그 자리를 대신한다. 스프링 부트는 `application.yml`의 설정을 읽어 JPA와 하이버네이트의 동작을 제어하는 똑똑한 조립자 역할을 한다.

이번 절에서는 실무에서 가장 중요하고 자주 사용되는 핵심 속성들을 정리한다. 이들을 정확히 이해하고 제어하는 것이 안정적인 JPA 애플리케이션의 첫걸음이다.

-----

### **1. 데이터소스(DataSource) 연결 속성**

가장 기본이 되는 데이터베이스 연결 정보. `spring.datasource.*` 경로에 설정한다.

  * **`url`**: 데이터베이스 접속 경로. (예: `jdbc:mysql://localhost:3306/mydb?useSSL=false&serverTimezone=Asia/Seoul`)
  * **`username`**: DB 접속 계정.
  * **`password`**: DB 접속 비밀번호. (운영 환경에서는 환경 변수나 Vault 같은 외부 시스템을 통해 주입하는 것이 안전하다.)
  * **`driver-class-name`**: JDBC 드라이버의 클래스 경로. (예: `com.mysql.cj.jdbc.Driver`)

-----

### **2. JPA 및 하이버네이트 핵심 제어 속성**

JPA와 그 구현체인 하이버네이트의 동작을 제어하는 가장 중요한 속성들이다.

  * **`spring.jpa.hibernate.ddl-auto`**

      * **역할**: 엔티티 매핑 정보를 바탕으로 애플리케이션 시작 시점에 데이터베이스 스키마를 어떻게 처리할지 결정한다.
      * `create`: 기존 테이블 삭제 후 다시 생성. **(로컬 개발용)** 데이터가 매번 초기화되므로 테스트에 용이하다.
      * `update`: 변경된 스키마만 찾아 반영. **(초기 개발용)** 편리하지만, 의도치 않은 변경을 유발할 수 있어 주의가 필요하다.
      * `validate`: 엔티티와 실제 테이블이 일치하는지만 검증. 불일치 시 애플리케이션이 시작되지 않는다. **(스테이징, 운영 환경 권장)**
      * `none`: 아무것도 하지 않음. **(운영 환경 권장)**
      * **경고**: **운영 환경에서 `create`나 `update`를 사용하는 것은 당신의 경력을 끝낼 수도 있는 재앙이다.**

  * **`spring.jpa.database-platform` (또는 `spring.jpa.properties.hibernate.dialect`)**

      * **역할**: JPA에게 현재 대화 상대가 누구인지 알려주는 '자기소개'와 같다. 하이버네이트는 이 설정을 보고 각 데이터베이스에 맞는 SQL 문법(방언, Dialect)을 사용한다. 페이징 쿼리 문법이 대표적인 예다.
      * **예시**: `org.hibernate.dialect.MySQLDialect`, `org.hibernate.dialect.H2Dialect`

  * **`spring.jpa.show-sql`**

      * **역할**: `true`로 설정하면, JPA가 실행하는 모든 SQL을 콘솔에 출력한다. 개발 시 필수적이지만, 단순히 문자열 조합이라 파라미터가 `?`로 표시되는 한계가 있다.

  * **`spring.jpa.properties.hibernate.format_sql`**

      * **역할**: `show-sql`이 `true`일 때, 출력되는 SQL을 보기 좋게 줄 바꿈 및 들여쓰기하여 포맷팅해준다. `show-sql`과 항상 함께 사용하는 것이 정신 건강에 이롭다.

  * **`spring.jpa.open-in-view`**

      * **역할**: OSIV(Open Session In View) 전략을 사용할지 결정한다. 기본값은 `true`지만, 이는 '편리함의 덫'이다. 데이터베이스 커넥션을 너무 오래 점유하여 심각한 성능 저하의 주범이 된다.
      * **권장**: **운영 환경에서는 반드시 `false`로 설정**하고, 서비스 계층에서 DTO 변환까지 모두 마치는 패턴을 사용해야 한다.

  * **`spring.jpa.properties.hibernate.default_batch_fetch_size`**

      * **역할**: 컬렉션이나 엔티티를 지연 로딩할 때 발생하는 N+1 문제의 가장 실용적인 해결사. 이 값을 설정하면, N번의 추가 쿼리가 `IN` 절을 사용하는 몇 개의 쿼리로 대폭 줄어든다.
      * **권장**: **100 \~ 1000 사이의 적절한 값으로 설정하는 것을 강력히 권장**한다. 이 설정 하나만으로 수많은 N+1 문제를 예방할 수 있다.

  * **`spring.jpa.properties.hibernate.use_sql_comments`**

      * **역할**: `true`로 설정하면, 실행되는 SQL 로그에 어떤 JPQL로부터 생성되었는지 주석을 함께 남겨준다. 복잡한 쿼리를 디버깅할 때 매우 유용하다.

-----

### **종합 예제 (`application.yml`)**

```yaml
# 공통 설정
spring:
  jpa:
    hibernate:
      ddl-auto: none # 기본값은 none으로 하여 안전하게 시작
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
        default_batch_fetch_size: 100
    open-in-view: false # OSIV는 끈다.

---

# 개발(local) 환경 프로필
spring:
  config:
    activate:
      on-profile: local
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa
    password:
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create # 개발 시에는 매번 새로 생성
    show-sql: true
  h2:
    console:
      enabled: true

---

# 운영(prod) 환경 프로필
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    # 운영 DB 정보는 외부에 설정하는 것이 좋다.
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: validate # 운영에서는 스키마 변경이 없음을 검증만 한다.
    database-platform: org.hibernate.dialect.MySQLDialect
```