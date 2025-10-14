## 04\. @Transient와 @Lob: JPA 관리 제외 대상과 대용량 데이터 처리

지금까지 우리는 엔티티의 모든 필드를 어떻게든 데이터베이스 컬럼에 매핑하는 방법을 다뤘다. 하지만 때로는 특정 필드를 의도적으로 매핑에서 제외해야 하거나, 일반적인 컬럼에는 담을 수 없을 만큼 큰 데이터를 저장해야 하는 경우도 발생한다. JPA는 이런 예외적인 상황들을 위한 `@Transient`와 `@Lob` 어노테이션을 제공한다.

### **`@Transient`: 이 필드는 DB와 상관없습니다**

엔티티 클래스에 정의된 필드라고 해서 모두 데이터베이스에 저장되어야 하는 것은 아니다. 예를 들어, 데이터베이스에 저장된 `firstName`과 `lastName`을 조합해서 임시로 `fullName`을 만들어 사용하거나, 비밀번호 확인처럼 메모리에서만 잠시 사용할 값이 필요할 수 있다.

이처럼 **데이터베이스에 저장하거나 조회하지 않고, 순수하게 객체 내에서만 사용하고 싶은 필드**가 있을 때 **`@Transient`** 어노테이션을 사용한다.

JPA는 `@Transient`가 붙은 필드를 완벽하게 무시한다. DDL 생성 시 해당 필드를 위한 컬럼을 만들지 않으며, `INSERT`나 `UPDATE` SQL에도 포함시키지 않고, 조회 시 값을 채워 넣으려고 시도하지도 않는다. 말 그대로 '투명 인간' 취급을 하는 것이다.

```kotlin
@Entity
class User(
    @Id @GeneratedValue
    val id: Long = 0,
    
    val firstName: String,
    
    val lastName: String,
    
    // 이 필드는 DB에 저장되지 않는다.
    @Transient
    val fullName: String = "$firstName $lastName"
)
```

위 예제에서 `fullName` 필드는 데이터베이스와는 아무런 관련이 없다. 그저 `User` 객체가 메모리에 존재하는 동안에만 `firstName`과 `lastName`을 조합하여 편의를 제공하는 역할을 할 뿐이다.

### **`@Lob`: 대용량 데이터를 위한 특별 지정석**

일반적으로 문자열은 `VARCHAR` 타입으로, 숫자는 `NUMBER`나 `BIGINT` 타입으로 매핑된다. 하지만 게시글 본문처럼 글자 수 제한이 없는 아주 긴 텍스트나, 이미지, 동영상 같은 바이너리 파일을 데이터베이스에 저장하려면 어떻게 해야 할까? `VARCHAR`의 최대 길이를 훌쩍 넘는 이런 데이터들을 위해 데이터베이스는 `CLOB`(Character Large Object)이나 `BLOB`(Binary Large Object) 같은 대용량 데이터 타입을 제공한다.

JPA에서는 **`@Lob`** 어노테이션을 사용하여 특정 필드를 이러한 대용량 타입과 매핑할 수 있다. `@Lob`은 Large Object의 약자다.

`@Lob`은 매핑하는 필드의 자바/코틀린 타입에 따라 저장되는 DB 타입이 달라진다.

  * **필드 타입이 `String`, `char[]`인 경우**: 데이터베이스의 **`CLOB`** 타입과 매핑된다. (MySQL의 `TEXT`, Oracle의 `CLOB`, PostgreSQL의 `TEXT` 등)
  * **필드 타입이 `byte[]`, `Byte[]`인 경우**: 데이터베이스의 **`BLOB`** 타입과 매핑된다. (MySQL의 `BLOB` or `LONGBLOB`, Oracle의 `BLOB`, PostgreSQL의 `bytea` 등)

<!-- end list -->

```kotlin
@Entity
@Table(name = "BOARD_POST")
class Post(
    @Id @GeneratedValue
    val id: Long = 0,
    
    @Column(length = 200, nullable = false)
    val title: String,
    
    // CLOB 타입으로 매핑되어 매우 긴 텍스트를 저장할 수 있다.
    @Lob
    @Column(nullable = false)
    val content: String,

    // BLOB 타입으로 매핑되어 이미지 등 바이너리 데이터를 저장할 수 있다.
    @Lob
    var attachment: ByteArray? = null
)
```

`@Lob`을 사용하면 `length` 속성을 지정할 필요가 없다. JPA가 알아서 해당 데이터베이스의 방언(Dialect)에 맞는 대용량 객체 타입으로 DDL을 생성해준다.

`@Transient`와 `@Lob`은 매일 사용하는 어노테이션은 아니지만, 이처럼 특별한 요구사항을 만났을 때 문제를 해결해주는 유용한 도구들이다. 이제 엔티티 매핑의 거의 마지막 단계로, 개발 환경과 운영 환경에서 `ddl-auto` 옵션을 어떻게 다루어야 하는지에 대한 중요한 이야기를 해보자.