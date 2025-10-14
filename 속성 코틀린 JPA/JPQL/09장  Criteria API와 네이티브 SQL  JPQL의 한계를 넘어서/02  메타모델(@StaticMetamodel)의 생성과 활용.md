## 02\. 메타모델(@StaticMetamodel)의 생성과 활용

Criteria의 유일한 장점은 타입-안전성이지만, `m.get<String>("username")` 처럼 필드 이름에 문자열을 사용하는 순간 그 장점마저 퇴색된다. "username"을 "usernmae"로 잘못 적어도 컴파일러는 잡아주지 못하고, 런타임 에러를 마주해야 한다.

JPA는 이 문제를 해결하기 위해 \*\*메타모델(Metamodel)\*\*이라는 기능을 제공한다. 메타모델은 엔티티 클래스에 대한 '메타데이터'를 담고 있는 별도의 클래스를 Annotation Processor를 통해 자동으로 생성하는 기능이다. 이 생성된 클래스를 사용하면, 쿼리에서 필드 이름을 문자열이 아닌, 타입-안전한 객체 참조로 사용할 수 있게 된다.

-----

### **1단계: 메타모델 생성 설정**

메타모델을 사용하려면 먼저 빌드 도구에 Annotation Processor 설정을 추가하여, 컴파일 시점에 메타모델 클래스가 자동으로 생성되도록 해야 한다.

**`build.gradle.kts` 설정**

```groovy
dependencies {
    // ...
    // 하이버네이트 JPA 메타모델 생성기 추가
    annotationProcessor("org.hibernate.orm:hibernate-jpamodelgen:6.4.4.Final")
}
```

이 설정을 추가하고 프로젝트를 빌드하면, `build/generated/sources/annotationProcessor/java/main` 경로에 `@Entity`가 붙은 모든 클래스에 대해 메타모델 클래스가 자동으로 생성된다. 예를 들어, `Member` 엔티티에 대해서는 `Member_` 라는 클래스가 생성된다.

**생성된 `Member_` 클래스 (예시)**

```java
// 코틀린이 아닌 자바로 생성됨
@StaticMetamodel(Member.class)
public abstract class Member_ {
    public static volatile SingularAttribute<Member, Long> id;
    public static volatile SingularAttribute<Member, String> username;
    public static volatile SingularAttribute<Member, Integer> age;
    public static volatile SingularAttribute<Member, Team> team;
}
```

`Member` 엔티티의 각 필드에 대응하는 `static` 필드가 생성된 것을 볼 수 있다. 이제 우리는 이 `Member_` 클래스를 Criteria 쿼리에서 사용할 수 있다.

-----

### **2단계: 메타모델을 활용한 Criteria 쿼리**

이전 절에서 작성했던 복잡한 Criteria 쿼리를 메타모델을 사용하여 타입-안전하게 개선해 보자.

```kotlin
// ...
val t = m.join(Member_.team) // 타입-안전한 조인

// m.get<String>("name") -> m.get(Member_.username) 으로 변경
val teamNamePredicate = cb.equal(t.get(Team_.name), "Team A")
val agePredicate = cb.greaterThanOrEqualTo(m.get(Member_.age), 30)

cq.select(m)
  .where(cb.and(teamNamePredicate, agePredicate))
  .orderBy(cb.desc(m.get(Member_.username)))
// ...
```

`m.get("username")` 처럼 문자열을 사용하던 부분이 `m.get(Member_.username)` 처럼 정적인 코드 참조로 바뀌었다. 이제 만약 `Member_.usernmae` 처럼 오타를 내면, IDE와 컴파일러가 즉시 오류를 알려준다. 타입 캐스팅(`get<String>`)이 필요 없어진 것도 장점이다.

-----

메타모델은 분명 Criteria 쿼리를 한층 더 견고하게 만들어준다. 하지만 근본적인 문제를 해결해주지는 못한다. 메타모델을 사용하더라도 Criteria 쿼리 특유의 **극단적인 복잡성과 낮은 가독성**은 전혀 개선되지 않는다. 코드는 여전히 길고, 복잡하고, 이해하기 어렵다.

결국 메타모델은 '이왕 Criteria를 쓰기로 했다면, 이렇게 쓰는 것이 그나마 가장 안전하다'는 지침을 줄 뿐, 'Criteria를 써야 하는 이유'가 되어주지는 못한다.

JPQL과 Criteria로도 해결할 수 없는 최후의 상황이 닥쳤을 때, 우리에게는 마지막 비상 탈출구가 남아있다. 다음 절에서 네이티브 SQL에 대해 알아보자.