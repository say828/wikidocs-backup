## 01\. @Id와 기본 키(Primary Key) 생성 전략: 정체성(Identity) 부여 방식 (IDENTITY, SEQUENCE, TABLE, AUTO)

현실 세계에서 모든 사람은 주민등록번호라는 고유한 식별자를 통해 구분된다. 데이터베이스의 세계에서도 마찬가지다. 수백만, 수천만 개의 로우(row) 속에서 특정 로우를 유일하게 식별할 수 있는 값, 그것이 바로 \*\*기본 키(Primary Key, PK)\*\*다. `@Entity`가 객체에게 영속성을 부여하는 출생신고라면, `@Id`는 그 객체에게 세상에 단 하나뿐인 주민등록번호를 부여하는 행위와 같다.

`@Id` 어노테이션은 엔티티 클래스 내의 특정 필드가 기본 키임을 JPA에게 알려주는 역할을 한다. 이 어노테이션이 붙은 필드를 \*\*식별자 필드(Identifier Field)\*\*라고 부른다.

```kotlin
@Entity
class Member(
    @Id // 이 필드가 바로 기본 키다.
    val id: Long = 0,
    var name: String
)
```

### **기본 키는 언제, 어떻게 할당되는가?**

그렇다면 이 `id` 값은 언제, 누가 만들어주는 걸까? 개발자가 직접 `member.id = 1L` 처럼 할당할 수도 있겠지만, 이는 매우 위험하고 번거로운 일이다. 동시성 문제로 같은 ID가 중복 저장될 수도 있고, 여러 테이블에 걸쳐 유일성을 보장하기도 어렵다.

다행히 JPA는 데이터베이스가 제공하는 다양한 자동 생성 방식을 활용하여 이 문제를 우아하게 해결한다. 바로 **`@GeneratedValue`** 어노테이션이다. 이 어노테이션은 식별자 필드에 적용하여 기본 키 값의 생성을 데이터베이스에 위임할 것임을 선언한다. 핵심은 `strategy` 속성이며, JPA 표준은 네 가지 전략을 제공한다.

[도표: JPA 기본 키 생성 전략 비교]

| 전략 (GenerationType) | 설명 | 지원 DB | 성능 | 특징 |
| --- | --- | --- | --- | --- |
| **IDENTITY** | 기본 키 생성을 DB에 완전히 위임. (예: MySQL의 `AUTO_INCREMENT`) | MySQL, PostgreSQL, SQL Server 등 | 중간 | `persist()` 시점에 즉시 `INSERT` 쿼리가 실행되고 DB에서 생성된 ID를 반환받음. **쓰기 지연 불가능.** |
| **SEQUENCE** | DB의 시퀀스 오브젝트를 사용하여 기본 키 할당. (예: Oracle, PostgreSQL) | Oracle, PostgreSQL, H2 등 | 좋음 | `persist()` 시점에 DB 시퀀스를 조회(`SELECT`)하여 ID를 미리 가져온 후, 트랜잭션 커밋 시 `INSERT` 실행. **쓰기 지연 가능.** |
| **TABLE** | 키 생성 전용 테이블을 만들어 DB 시퀀스를 흉내 내는 방식. | 모든 DB | 나쁨 | 테이블을 사용하므로 락(Lock)이 걸려 동시성 환경에서 성능 저하가 심각함. **거의 사용되지 않음.** |
| **AUTO** | 선택한 DB 방언(Dialect)에 따라 위 세 가지 전략 중 하나를 자동으로 선택. (기본값) | 모든 DB | DB에 따라 다름 | DB를 변경해도 코드를 수정할 필요가 없는 장점이 있지만, 개발자가 의도치 않은 전략으로 동작할 수 있으므로 명시하는 것을 권장. |

-----

### **1. IDENTITY 전략**

`IDENTITY` 전략은 가장 직관적이고 간단하다. 기본 키 생성을 데이터베이스에 완전히 맡긴다. MySQL의 `AUTO_INCREMENT`나 PostgreSQL의 `SERIAL` 컬럼이 대표적인 예다.

```kotlin
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
val id: Long = 0
```

이 전략의 가장 중요한 특징은 **엔티티가 `persist()` 되는 시점에 즉시 `INSERT` SQL이 데이터베이스로 전송된다**는 점이다. 왜 그럴까? `id` 값은 데이터베이스가 `INSERT`를 수행해야만 비로소 생성되기 때문이다. JPA는 영속성 컨텍스트에 엔티티를 저장할 때 반드시 식별자(ID) 값이 필요한데, `IDENTITY` 전략에서는 ID 값을 미리 알 수 없으므로, `persist()` 호출 즉시 DB와 통신하여 ID를 받아와야만 하는 것이다.

이 때문에 `IDENTITY` 전략은 JPA의 핵심 기능 중 하나인 **'트랜잭션을 지원하는 쓰기 지연'을 활용할 수 없다.** (이 주제는 2장에서 자세히 다룬다.)

-----

### **2. SEQUENCE 전략**

오라클, PostgreSQL 등 시퀀스 오브젝트를 지원하는 데이터베이스에서 주로 사용한다. 시퀀스는 유일한 값을 순서대로 생성하는 데이터베이스 오브젝트다.

```kotlin
@Entity
@SequenceGenerator(
    name = "MEMBER_SEQ_GENERATOR", // 시퀀스 생성기 이름
    sequenceName = "MEMBER_SEQ", // DB에 생성될 시퀀스 이름
    initialValue = 1, allocationSize = 50)
class Member(
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                    generator = "MEMBER_SEQ_GENERATOR") // 생성기 지정
    val id: Long = 0,
    //...
)
```

`@SequenceGenerator`를 통해 사용할 시퀀스 생성기를 등록하고, `@GeneratedValue`에서 해당 생성기를 지정하여 사용한다.

`SEQUENCE` 전략은 `IDENTITY`와 동작 방식이 다르다.

1.  `persist()`가 호출되면, JPA는 먼저 DB의 시퀀스에서 다음 ID 값을 조회하는 쿼리(`SELECT nextval('MEMBER_SEQ')`)를 실행한다.
2.  조회해 온 ID를 엔티티에 할당하고, 이 엔티티를 영속성 컨텍스트의 1차 캐시에 저장한다. 이 시점까지는 아직 `INSERT` 쿼리가 실행되지 않는다.
3.  이후 트랜잭션이 커밋될 때, 영속성 컨텍스트에 저장된 엔티티에 대한 `INSERT` 쿼리가 DB로 전송된다.

이처럼 ID를 미리 조회해 올 수 있기 때문에 **'쓰기 지연'이 가능**하다.

> **`allocationSize` 최적화**
> `allocationSize`는 성능 최적화를 위한 매우 중요한 속성이다. 만약 `allocationSize`가 50이라면, JPA는 시퀀스를 한 번 조회할 때 1부터 50까지의 ID를 미리 확보해두고 메모리에서 사용한다. 그리고 51번째 ID가 필요할 때 다시 시퀀스를 조회한다. 이렇게 하면 DB 통신 횟수를 1/50로 줄일 수 있어 성능이 크게 향상된다. **이 값을 `hibernate.id.new_generator_mappings=true` (스프링부트 2.0+ 기본값) 설정과 함께 잘 사용하는 것이 시퀀스 전략의 핵심이다.**

-----

### **3. TABLE 전략**

데이터베이스 시퀀스를 흉내 내는 전략이다. 키를 생성하는 용도의 테이블을 하나 만들고, 이 테이블의 특정 로우를 계속 업데이트하면서 ID를 생성한다.

```kotlin
@Entity
@TableGenerator(
    name = "MEMBER_SEQ_GENERATOR",
    table = "MY_SEQUENCES",
    pkColumnValue = "MEMBER_SEQ", allocationSize = 1)
class Member(
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE,
                    generator = "MEMBER_SEQ_GENERATOR")
    val id: Long = 0,
    //...
)
```

모든 데이터베이스에서 사용할 수 있다는 장점이 있지만, ID를 조회할 때마다 테이블에 `SELECT`와 `UPDATE`가 발생하고 락(Lock)이 걸리기 때문에 **성능이 매우 나쁘다.** 특별한 이유가 없다면 실무에서 사용할 일은 거의 없다.

-----

### **권장 전략**

어떤 전략을 선택해야 할까?

  * **`IDENTITY`**: 가장 간단하고 직관적이다. 쓰기 지연이 불가능하다는 단점이 있지만, 대부분의 웹 애플리케이션 환경에서는 큰 문제가 되지 않는 경우가 많다. MySQL을 사용한다면 자연스러운 선택이다.
  * **`SEQUENCE`**: `allocationSize`를 이용한 최적화가 가능하여 DB 통신 횟수를 줄일 수 있고, 쓰기 지연도 활용할 수 있어 성능상 이점이 있다. 오라클이나 PostgreSQL을 사용한다면 `SEQUENCE` 전략이 더 나은 선택이다.

이제 객체의 '주민등록번호'를 부여하는 방법을 배웠다. 다음 절에서는 객체의 일반 속성들을 테이블의 컬럼으로 매핑하는 `@Column` 어노테이션에 대해 자세히 알아보자.