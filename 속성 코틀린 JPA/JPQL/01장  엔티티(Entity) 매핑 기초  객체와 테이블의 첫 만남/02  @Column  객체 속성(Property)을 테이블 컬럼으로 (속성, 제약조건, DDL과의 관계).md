## 02\. @Column: 객체 속성(Property)을 테이블 컬럼으로 (속성, 제약조건, DDL과의 관계)

`@Id`를 통해 엔티티의 식별자를 매핑했다면, 이제 나머지 일반 속성들을 테이블의 컬럼과 연결할 차례다. 이때 사용하는 가장 기본적이고 중요한 어노테이션이 바로 \*\*`@Column`\*\*이다. 사실 `@Column` 어노테이션을 생략해도 대부분의 필드는 JPA가 알아서 잘 매핑해 준다. 자바/코틀린의 기본 타입들은 적절한 데이터베이스 컬럼 타입으로 자동 변환되고, 필드 이름을 기반으로 컬럼 이름도 결정된다(예: `userName` -\> `user_name`).

그렇다면 `@Column`은 언제 사용할까? 바로 이 **자동 매핑 규칙을 벗어나 더 세밀한 제어**가 필요할 때 사용한다. 예를 들어, 객체의 필드 이름과 테이블의 컬럼 이름을 다르게 하고 싶거나, `NOT NULL` 이나 `UNIQUE` 같은 제약조건을 걸고 싶거나, 문자열의 길이를 제한하고 싶을 때 `@Column`의 다양한 속성들이 강력한 힘을 발휘한다.

### **@Column의 핵심 속성들**

`@Column`은 여러 속성을 제공하며, 이들은 데이터베이스 스키마(DDL) 생성과 런타임 동작 모두에 영향을 미친다.

  * `name`: 객체 필드와 매핑할 테이블 컬럼의 이름을 직접 지정한다. 데이터베이스의 명명 규칙이 애플리케이션의 명명 규칙과 다를 때 유용하다.

```kotlin
@Column(name = "USER_NM")
var name: String
```

  * `nullable`: 컬럼의 `NULL` 허용 여부를 설정한다. `false`로 설정하면 DDL 생성 시 `NOT NULL` 제약조건이 추가된다. 기본값은 `true`다. **코틀린의 Null-Safe 타입과 완벽한 조화를 이룬다.** `var name: String` 처럼 non-null 타입으로 선언했다면, `@Column(nullable = false)`를 함께 명시하여 애플리케이션 모델과 데이터베이스 스키마의 정합성을 맞추는 것이 바람직하다.

```kotlin
@Column(nullable = false)
var email: String // null이 될 수 없는 타입

var address: String? // null이 될 수 있는 타입
```

  * `unique`: 컬럼에 `UNIQUE` 제약조건을 설정한다. DDL 생성 시 유니크 제약조건이 추가된다. 하지만 이 방식은 제약조건의 이름을 지정할 수 없어 관리가 어렵다. 둘 이상의 컬럼을 묶는 복합 유니크 키는 설정이 불가능하다. 따라서 더 나은 방법은 이전 절에서 배운 `@Table`의 `uniqueConstraints` 속성을 사용하는 것이다.

  * `length`: 문자열 타입 컬럼의 길이를 지정한다. `String` 타입 필드에만 사용된다. DDL 생성 시 `VARCHAR(지정된 길이)` 로 생성된다. 기본값은 보통 255다. 이 값을 적절히 설정하면 데이터베이스 공간을 효율적으로 사용하고, 데이터 길이에 대한 암묵적인 검증 효과도 얻을 수 있다.

```kotlin
@Column(length = 20)
var nickname: String
```

  * `insertable`, `updatable`: 각각 이 컬럼이 `INSERT` 문과 `UPDATE` 문에 포함될지 여부를 결정한다. 기본값은 둘 다 `true`다. 예를 들어, 최초 등록 시간처럼 한 번 정해지면 절대 바뀌어서는 안 되는 값에는 `@Column(updatable = false)`를 설정하여 실수를 방지할 수 있다.

```kotlin
@Column(updatable = false)
val createdAt: LocalDateTime = LocalDateTime.now()
```

  * `columnDefinition`: 특정 데이터베이스에 종속적인 컬럼 속성을 직접 정의하고 싶을 때 사용한다. 예를 들어, `default` 값을 설정하거나 특정 DB만의 데이터 타입을 사용하고 싶을 때 유용하다. 하지만 이 속성을 사용하면 데이터베이스 이식성이 떨어지므로 신중하게 사용해야 한다.

```kotlin
// DDL 생성 시 'status VARCHAR(10) default 'ACTIVE'' 와 같은 구문이 추가됨
@Column(columnDefinition = "VARCHAR(10) default 'ACTIVE'")
var status: String = "ACTIVE"
```

  * `precision`, `scale`: `BigDecimal`과 같은 아주 정밀한 소수점 값을 다룰 때 사용한다. `precision`은 전체 자릿수를, `scale`은 소수점 이하 자릿수를 의미한다. 금액이나 정밀 계산이 필요한 값에 필수적이다.

```kotlin
// 총 19자리, 소수점 이하 2자리의 숫자를 표현 (예: 12345678901234567.12)
@Column(precision = 19, scale = 2)
var balance: BigDecimal
```

### **DDL과의 관계: 개발과 운영의 차이**

`nullable`, `length`, `unique` 같은 속성들은 `spring.jpa.hibernate.ddl-auto` 옵션이 `create`나 `update`로 설정된 개발 환경에서는 매우 유용하다. JPA가 엔티티 정의를 보고 알아서 데이터베이스 스키마를 생성해주기 때문이다.

하지만 실제 운영 환경에서는 `ddl-auto`를 `none`이나 `validate`로 설정하고, Flyway나 Liquibase 같은 도구를 사용해 별도의 DDL 스크립트로 스키마를 관리하는 것이 일반적이다. 이 경우 `@Column`의 제약조건 관련 속성들은 DB 스키마에 직접적인 영향을 주지 않는다.

그렇다고 해서 이 속성들이 쓸모없어지는 것은 아니다. 이들은 여전히 다음과 같은 중요한 역할을 한다.

1.  **애플리케이션 코드 레벨의 문서화**: 엔티티 클래스만 봐도 해당 컬럼의 제약조건을 명확히 알 수 있다.
2.  **잠재적인 유효성 검증**: JPA 구현체가 런타임에 이 정보를 활용하여 특정 최적화를 수행하거나, 일부 상황에서 예외를 발생시킬 수 있다.

따라서 운영 환경에서 DDL을 수동으로 관리하더라도, 엔티티의 `@Column` 설정과 실제 데이터베이스 스키마는 **항상 일관되게 유지**하는 것이 가장 이상적인 베스트 프랙티스다. 이제 일반적인 타입 매핑을 넘어, 조금 더 특별한 타입들을 다루는 방법을 다음 절에서 알아보자.