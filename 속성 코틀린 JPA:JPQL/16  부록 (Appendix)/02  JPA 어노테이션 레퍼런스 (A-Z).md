## 02. JPA 어노테이션 레퍼런스 (A-Z)

이 절은 본문에서 다룬 핵심적인 JPA 어노테이션들을 빠르게 찾아볼 수 있도록 정리한 요약본이다. 알파벳 순서보다는 기능별로 그룹화하여 더 쉽게 찾아볼 수 있도록 구성했다. 각 어노테이션의 가장 중요한 역할과 핵심 속성을 중심으로 설명한다.

---

### **1. 엔티티 및 테이블 매핑**

* **`@Entity`**
    * **역할**: 이 클래스가 JPA가 관리하는 엔티티임을 선언한다. 가장 기본적이고 필수적인 어노테이션이다.
    * **핵심 속성**: `name` (JPQL에서 사용할 엔티티 이름을 지정. 기본값은 클래스 이름)

* **`@Table`**
    * **역할**: 엔티티와 매핑될 데이터베이스 테이블의 세부 정보를 지정한다.
    * **핵심 속성**: `name` (매핑할 테이블 이름), `uniqueConstraints` (유니크 제약조건 지정)

* **`@MappedSuperclass`**
    * **역할**: 엔티티는 아니지만, 여러 엔티티가 공통으로 사용하는 매핑 정보(필드)를 담고 있는 부모 클래스임을 선언한다. 테이블은 생성되지 않고, 자식 엔티티의 테이블에 해당 필드들이 컬럼으로 추가된다.

---

### **2. 기본 키 매핑**

* **`@Id`**
    * **역할**: 엔티티의 식별자, 즉 기본 키(Primary Key) 필드임을 나타낸다.

* **`@GeneratedValue`**
    * **역할**: 기본 키의 값을 어떻게 생성할지 전략을 지정한다.
    * **핵심 속성**: `strategy`
        * `IDENTITY`: DB에 키 생성을 위임 (MySQL `AUTO_INCREMENT` 등). `persist` 시 즉시 INSERT 실행.
        * `SEQUENCE`: DB 시퀀스 오브젝트를 사용 (Oracle 등). `allocationSize`로 성능 최적화 가능.
        * `TABLE`: 키 생성 전용 테이블을 사용. 성능 문제로 거의 사용 안 함.
        * `AUTO`: DB 방언에 따라 위 세 가지 중 하나를 자동으로 선택 (기본값).

---

### **3. 필드 및 컬럼 매핑**

* **`@Column`**
    * **역할**: 객체의 필드를 테이블의 컬럼에 매핑할 때 세부 속성을 지정한다.
    * **핵심 속성**: `name` (컬럼명), `nullable` (`false`로 지정 시 `NOT NULL` 제약조건), `unique` (유니크 제약조건), `length` (문자열 길이), `updatable`, `insertable`, `columnDefinition` (직접 DDL 구문 정의)

* **`@Enumerated`**
    * **역할**: Enum 타입을 어떻게 매핑할지 지정한다.
    * **핵심 속성**: `value`
        * `EnumType.STRING`: Enum의 이름(문자열)으로 저장. **(강력히 권장)**
        * `EnumType.ORDINAL`: Enum의 순서(숫자)로 저장. (데이터 깨질 위험이 커 절대 사용 금지)

* **`@Lob`**
    * **역할**: 필드를 데이터베이스의 대용량 객체(Large Object) 타입과 매핑한다. `String`은 `CLOB`으로, `byte[]`는 `BLOB`으로 매핑된다.

* **`@Transient`**
    * **역할**: 이 필드는 데이터베이스와 아무 관련이 없음을 선언한다. 즉, 영속성 관리 대상에서 제외된다.

* **`@Embeddable` / `@Embedded`**
    * **역할**: 값 타입(Value Type)을 정의하고 사용하기 위한 어노테이션 쌍. `@Embeddable`은 값 타입 클래스에, `@Embedded`는 그 클래스를 사용하는 필드에 붙인다.

---

### **4. 연관관계 매핑**

* **`@ManyToOne` / `@OneToMany` / `@OneToOne` / `@ManyToMany`**
    * **역할**: 연관관계의 종류(Cardinality)를 정의한다. `@ManyToMany`는 실무에서 사용하지 않는 것이 좋다.
    * **핵심 속성**: `fetch` (`FetchType.LAZY` 또는 `EAGER`), `cascade` (영속성 전이 옵션), `optional`, `orphanRemoval` (`true`로 지정 시 고아 객체 제거 기능 활성화)

* **`@JoinColumn`**
    * **역할**: 연관관계의 주인 쪽에서 외래 키(FK)를 매핑할 때 사용한다.
    * **핵심 속성**: `name` (외래 키 컬럼명), `referencedColumnName` (상대 테이블이 참조할 컬럼명. 보통 PK이므로 생략)

* **`@OneToMany`의 `mappedBy`**
    * **역할**: 연관관계의 주인이 아닌 쪽에서, 자신이 무엇에 의해 매핑되었는지(주인이 누구인지)를 명시한다. 값으로는 주인 쪽 엔티티에 있는 연관관계 필드명을 적는다.

---

### **5. 상속 관계 매핑**

* **`@Inheritance`**
    * **역할**: 부모 엔티티 클래스에 사용하여, 상속 관계를 테이블에 어떻게 매핑할지 전략을 지정한다.
    * **핵심 속성**: `strategy`
        * `InheritanceType.JOINED`: 조인 전략 (정규화)
        * `InheritanceType.SINGLE_TABLE`: 단일 테이블 전략 (성능)
        * `InheritanceType.TABLE_PER_CLASS`: 클래스별 테이블 전략 (비권장)

* **`@DiscriminatorColumn` / `@DiscriminatorValue`**
    * **역할**: `JOINED`나 `SINGLE_TABLE` 전략에서, 해당 로우가 어떤 자식 엔티티에 해당하는지 구분하기 위한 컬럼(`@DiscriminatorColumn`)과 그 값(`@DiscriminatorValue`)을 지정한다.

---

### **6. 고급 기능 및 기타**

* **`@Version`**
    * **역할**: 낙관적 락(Optimistic Lock)을 위한 버전 관리 필드에 사용한다.

* **`@EntityListeners`**
    * **역할**: 엔티티의 생명주기 이벤트를 처리할 외부 리스너 클래스를 지정한다. Spring Data JPA의 Auditing 기능에서 핵심적으로 사용된다.
* **`@Query`, `@Param` (Spring Data JPA)**
    * **역할**: 리포지토리 메서드에 JPQL이나 네이티브 SQL을 직접 바인딩한다. `@Param`은 이름 기반 파라미터를 매핑한다.
* **`@EntityGraph` (Spring Data JPA)**
    * **역할**: JPQL 수정 없이, 쿼리 실행 시점에 동적으로 페치 조인할 대상을 지정한다. N+1 문제 해결에 유용하다.