## 03\. 프로젝션(Projection): 엔티티, 임베디드 타입, 스칼라 타입 조회

`SELECT m FROM Member m` 처럼 엔티ティ 전체를 조회하는 경우도 많지만, 실제 애플리케이션에서는 특정 데이터만 골라서 보고 싶을 때가 더 많다. 예를 들어, 모든 회원의 '이름'만 목록으로 가져오거나, '이름과 나이'를 함께 조회하는 경우다. 이처럼 `SELECT` 절에 조회할 대상을 명시적으로 지정하는 것을 \*\*프로젝션(Projection)\*\*이라고 한다. 🛰️

JPQL의 프로젝션 대상은 크게 세 가지 종류로 나눌 수 있다.

-----

### **1. 엔티티 프로젝션 (Entity Projection)**

`SELECT m FROM Member m` 처럼 `SELECT` 절에 엔티티 자체를 지정하는 것이다. 이미 앞에서 다룬 내용이지만, 중요한 특징을 다시 한번 짚고 넘어가자.

```jpql
SELECT m FROM Member m
```

  * **반환 타입**: `TypedQuery<Member>`
  * **결과**: `List<Member>`
  * **특징**: 여기서 조회된 `Member` 객체들은 **영속성 컨텍스트에 의해 관리**된다. 따라서 이후에 해당 객체의 값을 변경하면 트랜잭션 커밋 시점에 변경 감지가 일어나 `UPDATE` 쿼리가 자동으로 실행된다.

엔티티 프로젝션은 연관된 엔티티를 조회할 때도 유용하다. `JOIN`을 통해 연관된 엔티티를 함께 프로젝션할 수 있다.

```jpql
-- Member와 연관된 Team을 함께 조회한다.
SELECT m.team FROM Member m
```

이 쿼리는 모든 `Member`가 속한 `Team` 엔티티들을 조회한다.

-----

### **2. 임베디드 타입 프로젝션 (Embedded Type Projection)**

`@Embeddable`로 정의한 값 타입을 프로젝션하는 것도 가능하다. `Member`가 `Address`라는 임베디드 타입을 가지고 있다고 가정해 보자.

```jpql
-- 모든 회원의 주소 정보만 조회한다.
SELECT m.address FROM Member m
```

  * **반환 타입**: `TypedQuery<Address>`
  * **결과**: `List<Address>`
  * **특징**: 여기서 조회된 `Address` 객체는 \*\*값 타입(Value Type)\*\*이므로 엔티티가 아니다. 따라서 영속성 컨텍스트에 의해 관리되지 않는다.

임베디드 타입 프로젝션은 엔티티의 일부 속성을 응집력 있게 묶어 조회할 때 유용하다.

-----

### **3. 스칼라 타입 프로젝션 (Scalar Type Projection)**

가장 흔하게 사용되는 프로젝션 방식으로, 엔티티의 특정 필드들(숫자, 문자열, 날짜 등과 같은 기본 데이터 타입)만 골라서 조회하는 것이다. 스칼라(Scalar)는 '단일 값'을 의미한다.

```jpql
-- 모든 회원의 '이름'만 조회한다.
SELECT m.username FROM Member m
```

  * **반환 타입**: `TypedQuery<String>`
  * **결과**: `List<String>`

둘 이상의 필드를 조회할 수도 있다.

```jpql
-- 모든 회원의 '이름'과 '나이'를 조회한다.
SELECT m.username, m.age FROM Member m
```

이 경우, JPQL은 여러 타입의 값들을 반환하므로 `TypedQuery`를 특정 클래스로 지정할 수 없다. 결과를 받는 방법은 두 가지다.

1.  **`Query` 타입으로 조회 후 직접 캐스팅**

    ```kotlin
    val query = entityManager.createQuery("SELECT m.username, m.age FROM Member m")
    val resultList: List<Any?> = query.resultList // List<Object[]> 또는 코틀린에서는 List<Any?>
    for (row in resultList) {
        val rowArray = row as Array<Any>
        val username = rowArray[0] as String
        val age = rowArray[1] as Int
        println("username = $username, age = $age")
    }
    ```

    코드가 지저분하고 타입 캐스팅이 번거롭다는 단점이 있다.

2.  **`Object[]` 타입으로 조회 (더 명시적)**

    ```kotlin
    val query = entityManager.createQuery("SELECT m.username, m.age FROM Member m", Array<Any>::class.java)
    val resultList: List<Array<Any>> = query.resultList
    // ... 이후 로직은 위와 동일
    ```

    반환 타입을 `Array<Any>`(자바에서는 `Object[]`)로 명시하여 조금 더 타입 안전하게 결과를 받을 수 있다.

하지만 이 두 가지 방법 모두 여전히 불편하다. 조회 결과를 객체 배열이 아닌, 의미 있는 DTO(Data Transfer Object) 객체로 직접 받아올 수는 없을까? 다음 절에서 이 문제를 해결하는 가장 실용적인 방법을 알아본다.