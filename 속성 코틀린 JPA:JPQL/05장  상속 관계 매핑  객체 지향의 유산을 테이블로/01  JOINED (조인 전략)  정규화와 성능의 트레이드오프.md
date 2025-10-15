## 01\. JOINED (조인 전략): 정규화와 성능의 트레이드오프

\*\*조인 전략(Joined Strategy)\*\*은 상속 관계를 표현하는 가장 교과서적이고 정석적인 방법이다. 이 전략은 데이터베이스 설계의 **정규화(Normalization)** 원칙에 가장 충실하다. 객체의 상속 구조와 유사하게, 부모와 자식 테이블을 분리하고 이들을 조인하여 관계를 표현한다.

-----

### **매핑 코드와 테이블 구조**

조인 전략을 사용하려면, 부모 클래스에 `@Inheritance(strategy = InheritanceType.JOINED)` 어노테이션을 명시해야 한다.

**Item.kt (부모)**

```kotlin
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "DTYPE") // 자식 테이블을 구분하는 컬럼 (선택사항이지만 권장)
abstract class Item(
    @Id @GeneratedValue
    val id: Long = 0,
    var name: String,
    var price: Int
)
```

**Book.kt (자식)**

```kotlin
@Entity
// @DiscriminatorValue("B") // DTYPE 컬럼에 저장될 값. 기본값은 엔티티 이름("Book")
class Book(
    var author: String,
    var isbn: String,
) : Item()
```

**Album.kt (자식)**

```kotlin
@Entity
// @DiscriminatorValue("A")
class Album(
    var artist: String,
) : Item()
```

  * `@Inheritance(strategy = InheritanceType.JOINED)`: 상속 매핑 전략으로 조인 전략을 사용하겠다고 선언한다.
  * `@DiscriminatorColumn`: 부모 테이블(`ITEM`)에 추가되는 구분 컬럼이다. 이 컬럼에는 자식 엔티티의 종류(예: 'Book', 'Album')가 저장된다. 이를 통해 특정 `ITEM`이 `Book`인지 `Album`인지 구분할 수 있다. 필수는 아니지만, 어떤 종류의 자식과 조인해야 할지 명확히 알 수 있어 성능상 이점이 있다.
  * `@DiscriminatorValue`: 구분 컬럼(`DTYPE`)에 저장될 값을 직접 지정한다. 지정하지 않으면 기본적으로 엔티티 클래스 이름이 사용된다.

**생성되는 테이블 구조**
[그림: 조인 전략의 테이블 구조 (ITEM, BOOK, ALBUM)]
이 매핑을 통해 JPA는 다음과 같은 세 개의 테이블을 생성한다.

**ITEM (부모 테이블)**
| ID (PK) | DTYPE | NAME | PRICE |
| :--- | :--- | :--- | :--- |
| 1 | Book | JPA 정복 | 35000 |
| 2 | Album| BTS 앨범 | 28000 |

**BOOK (자식 테이블)**
| ID (PK, FK) | AUTHOR | ISBN |
| :--- | :--- | :--- |
| 1 | 김영한 | 123-456 |

**ALBUM (자식 테이블)**
| ID (PK, FK) | ARTIST |
| :--- | :--- |
| 2 | BTS |

자식 테이블의 `ID` 컬럼은 자기 자신의 \*\*기본 키(PK)\*\*이면서 동시에 부모 테이블의 `ID`를 참조하는 **외래 키(FK)** 역할을 한다.

-----

### **장점과 단점**

**장점:**

1.  **데이터 정규화**: 테이블이 정규화되어 있어 데이터 중복이 최소화되고, 저장 공간을 효율적으로 사용한다.
2.  **명확한 스키마**: 객체 상속 구조가 데이터베이스 스키마에 직관적으로 반영되어 이해하기 쉽다.
3.  **외래 키 제약조건**: `INSERT`나 `UPDATE` 시 외래 키 제약조건을 활용한 데이터 무결성을 보장할 수 있다.

**단점:**

1.  **조회 성능**: 데이터를 조회할 때 **반드시 조인(JOIN)이 발생**한다. 상속 구조가 복잡하고 깊어질수록 조인해야 할 테이블이 늘어나 성능이 저하될 수 있다.
2.  **쿼리 복잡성**: 데이터를 등록할 때는 `ITEM`과 `BOOK` 두 테이블에 각각 `INSERT` 쿼리가 실행되는 등, 쿼리가 상대적으로 복잡하다.

-----

조인 전략은 데이터의 정합성과 명확한 구조를 중시하는 설계에 적합하다. 하지만 조회 성능이 매우 중요한 비즈니스 로직에서는 조인으로 인한 비용을 반드시 고려해야 한다. 이제 조인 전략과 정반대의 특성을 가진 단일 테이블 전략에 대해 다음 절에서 알아보자.