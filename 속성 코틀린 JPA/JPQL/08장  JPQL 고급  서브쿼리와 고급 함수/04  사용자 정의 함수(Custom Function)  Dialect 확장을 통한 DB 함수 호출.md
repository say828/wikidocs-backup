## 04\. 사용자 정의 함수(Custom Function): Dialect 확장을 통한 DB 함수 호출

JPQL의 내장 함수는 표준적이고 이식성이 높다는 장점이 있지만, 때로는 우리가 사용하는 특정 데이터베이스가 제공하는 강력한 고유 함수를 사용하고 싶을 때가 있다. 예를 들어, MySQL의 `GROUP_CONCAT` 함수는 그룹화된 결과의 문자열을 하나로 합쳐주는 매우 유용한 기능이지만, JPQL 표준에는 포함되어 있지 않다.

이처럼 **JPA 표준에는 없지만, 특정 데이터베이스에서 제공하는 함수를 JPQL에서 사용하고 싶을 때** 우리는 \*\*사용자 정의 함수(Custom Function)\*\*를 등록하여 호출할 수 있다.

-----

### **Dialect 확장을 통한 함수 등록**

이 과정의 핵심은 하이버네이트의 **방언(Dialect)** 클래스를 확장하는 것이다. Dialect는 각 데이터베이스의 고유한 SQL 문법과 함수 차이를 하이버네이트가 흡수하고 처리하는 역할을 하는 클래스다. 우리는 사용하려는 데이터베이스의 Dialect를 상속받는 새로운 클래스를 만들고, 거기에 우리가 사용할 함수를 등록하면 된다.

**1단계: Custom Dialect 클래스 생성**
`MySQL`을 기준으로, 문자열을 합치는 `group_concat` 함수를 등록하는 예제를 만들어 보자.

**CustomMysqlDialect.kt**

```kotlin
package com.masterclass.jpamaster.config

import org.hibernate.dialect.MySQLDialect
import org.hibernate.dialect.function.StandardSQLFunction
import org.hibernate.type.StandardBasicTypes

class CustomMysqlDialect : MySQLDialect() {
    init {
        // registerFunction("JPQL에서 사용할 이름", SQL 함수)
        // 여기서는 GROUP_CONCAT 함수를 group_concat 이라는 이름으로 등록한다.
        registerFunction(
            "group_concat",
            StandardSQLFunction("GROUP_CONCAT", StandardBasicTypes.STRING)
        )
    }
}
```

`MySQLDialect`를 상속받아, 생성자(`init` 블록)에서 `registerFunction()` 메서드를 호출하여 `GROUP_CONCAT`이라는 SQL 함수를 `group_concat`이라는 이름으로 JPQL에 등록했다.

**2단계: `application.yml`에 Custom Dialect 설정**
이제 JPA(하이버네이트)가 기본 Dialect 대신 우리가 만든 Custom Dialect를 사용하도록 설정해야 한다.

**application.yml**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        # hibernate.dialect 속성에 우리가 만든 클래스의 전체 경로를 지정한다.
        dialect: com.masterclass.jpamaster.config.CustomMysqlDialect
```

**3단계: JPQL에서 `function()` 키워드로 호출**
모든 설정이 끝났다. 이제 JPQL에서 `function()`이라는 키워드를 사용하여 우리가 등록한 함수를 호출할 수 있다.

**JPQL 문법**: `function('등록한 함수 이름', 인자1, 인자2, ...)`

```jpql
-- 각 팀(Team)별로 소속된 회원(Member)들의 이름을 쉼표로 구분하여 한 줄로 조회
SELECT t.name, function('group_concat', m.username)
FROM Team t JOIN t.members m
GROUP BY t.name
```

이 JPQL은 하이버네이트에 의해 다음과 같은 MySQL 전용 SQL로 정상적으로 번역되어 실행된다.

```sql
SELECT t.name, GROUP_CONCAT(m.username)
FROM team t JOIN member m ON t.id = m.team_id
GROUP BY t.name
```

-----

사용자 정의 함수는 JPQL의 기능을 데이터베이스 레벨까지 확장시켜주는 매우 강력한 도구다. 하지만 이 기능을 사용하는 순간, 해당 JPQL은 특정 데이터베이스에 종속되게 되어 **이식성을 잃는다**는 점을 반드시 명심해야 한다. `group_concat`을 사용한 JPQL은 더 이상 Oracle이나 PostgreSQL에서는 동작하지 않는다. 따라서 데이터베이스를 변경할 가능성이 거의 없고, 해당 함수의 사용이 주는 이점이 이식성을 포기할 만큼 클 때 신중하게 사용해야 한다.

이제 JPQL로 데이터를 조회하는 거의 모든 기술을 배웠다. 마지막으로, 조회(SELECT)가 아닌, 대량의 데이터를 한 번에 수정(UPDATE)하거나 삭제(DELETE)하는 벌크 연산에 대해 알아보자.