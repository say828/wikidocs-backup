## 01\. 다대다(@ManyToMany)의 한계와 실무적 대안: 연결 테이블(@OneToMany + @ManyToOne)

객체 세상에는 다대다 관계가 분명히 존재한다. 학생과 강의의 관계를 생각해 보자. 학생 한 명은 여러 강의를 수강할 수 있고, 강의 하나에는 여러 학생이 수강한다. 이 관계를 JPA의 `@ManyToMany`를 사용하면 아주 간단하게 매핑할 수 있을 것처럼 보인다.

```kotlin
// 절대 이렇게 사용하지 말 것!
@Entity
class Student(
    //...
    @ManyToMany
    @JoinTable(name = "STUDENT_COURSE",
        joinColumns = [JoinColumn(name = "STUDENT_ID")],
        inverseJoinColumns = [JoinColumn(name = "COURSE_ID")])
    var courses: MutableList<Course> = mutableListOf()
)

@Entity
class Course(
    //...
    @ManyToMany(mappedBy = "courses")
    var students: MutableList<Student> = mutableListOf()
)
```

`@JoinTable` 어노테이션을 사용하면 `STUDENT_COURSE`라는 중간 연결 테이블까지 JPA가 자동으로 생성해준다. 코드는 간결하고, 객체지향적으로도 완벽해 보인다. 하지만 이 편리함 뒤에는 **실무에서 `@ManyToMany`를 절대로 사용해서는 안 되는 치명적인 한계**가 숨어있다.

-----

### **`@ManyToMany`의 치명적인 한계: 연결 테이블의 실종**

문제의 핵심은 **연결 테이블(`STUDENT_COURSE`)이 엔티티로 매핑되지 않고 JPA에 의해 숨겨져 있다는 점**이다. 이 때문에 우리는 연결 테이블에 대한 어떠한 추가 정보도 담을 수 없다.

현실 세계의 다대다 관계는 단순한 연결로 끝나지 않는 경우가 대부분이다. 학생이 강의를 수강하면, 그 관계에는 '수강 신청일', '성적'과 같은 추가적인 데이터가 반드시 필요하다. 쇼핑몰의 고객과 상품 관계를 생각해 봐도 마찬가지다. 중간 테이블인 `ORDERS`에는 '주문 수량', '주문 가격', '주문 시간' 등 핵심적인 정보가 들어간다.

`@ManyToMany`를 사용하면 이처럼 중요한 추가 컬럼들을 매핑할 방법이 원천적으로 차단된다. JPA가 자동으로 생성한 연결 테이블에는 두 엔티티의 ID 값 외에는 아무것도 추가할 수 없기 때문이다. 이 한계점 하나만으로도 `@ManyToMany`는 실무에서 사용할 가치를 잃는다.

-----

### **실무적 대안: 연결 엔티티(Connecting Entity)를 사용한 분해**

그렇다면 이 문제를 어떻게 해결해야 할까? 정답은 **다대다 관계를 두 개의 일대다 관계로 분해**하는 것이다. 중간에 **연결 엔티티**를 명시적으로 만들어서, 이 엔티티가 양쪽 엔티티와 일대다, 다대일 관계를 맺도록 설계하는 방식이다.

**1. 연결 엔티티 `StudentCourse` 생성**

```kotlin
@Entity
class StudentCourse(
    @Id @GeneratedValue
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "STUDENT_ID")
    val student: Student,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "COURSE_ID")
    val course: Course,

    // 추가적인 컬럼들을 자유롭게 매핑할 수 있다!
    val registrationDate: LocalDateTime,
    var grade: String? = null
)
```

  * 연결 테이블을 `StudentCourse`라는 독립적인 엔티티로 승격시켰다.
  * 이제 이 엔티티에 `registrationDate`, `grade`와 같은 비즈니스에 필요한 컬럼을 얼마든지 추가할 수 있다.

**2. 기존 엔티티의 관계 수정**

```kotlin
@Entity
class Student(
    //...
    @OneToMany(mappedBy = "student")
    var studentCourses: MutableList<StudentCourse> = mutableListOf()
)

@Entity
class Course(
    //...
    @OneToMany(mappedBy = "course")
    var studentCourses: MutableList<StudentCourse> = mutableListOf()
)
```

  * 기존의 `@ManyToMany`를 모두 제거한다.
  * `Student`와 `Course`는 각각 `StudentCourse`와 일대다(@OneToMany) 관계를 맺는다.

이 설계는 `@ManyToMany`의 모든 단점을 해결한다. 우리는 연결 테이블을 완벽하게 제어할 수 있게 되었고, 필요에 따라 얼마든지 확장할 수 있다. 쿼리 또한 `@ManyToMany`를 사용할 때보다 훨씬 더 명시적이고 예측 가능하게 동작한다.

> **결론: `@ManyToMany`는 잊어라.**
> 실무에서 다대다 관계를 매핑해야 한다면, 고민할 필요 없이 **연결 엔티티를 사용하는 일대다-다대일 관계로 분해**하는 것이 유일한 정답이다. `@ManyToMany`는 개념 증명을 위한 장난감일 뿐, 실제 운영 환경의 복잡성을 감당할 수 없다.

이제 관계 매핑의 가장 큰 지뢰를 제거하는 법을 배웠다. 다음 절에서는 엔티티의 생명주기를 연관된 엔티티에 전파하는 강력한 기능인 '영속성 전이(Cascade)'에 대해 알아보자.