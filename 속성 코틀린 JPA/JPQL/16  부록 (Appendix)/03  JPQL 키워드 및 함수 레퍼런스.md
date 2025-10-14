## 03. JPQL 키워드 및 함수 레퍼런스

이 절은 JPQL을 작성할 때 자주 사용되는 핵심 키워드와 내장 함수를 빠르게 찾아볼 수 있도록 정리한 요약본이다. 각 키워드의 역할과 기본적인 사용법을 중심으로 설명한다.

---

### **1. 기본 쿼리 절**

* **`SELECT`**
    * **역할**: 조회할 대상을 지정한다. 엔티티, 임베디드 타입, 스칼라 값 등 다양한 대상을 프로젝션할 수 있다.
    * **예시**: `SELECT m FROM Member m`, `SELECT m.username, m.age FROM Member m`

* **`FROM`**
    * **역할**: 쿼리의 대상을 지정하는 엔티티를 선언한다. 테이블 이름이 아닌 엔티티 이름을 사용하며, **반드시 별칭(alias)을 지정**해야 한다.
    * **예시**: `FROM Member m`

* **`WHERE`**
    * **역할**: 조회할 결과를 필터링하는 조건을 지정한다.
    * **예시**: `WHERE m.age > 20 AND m.team IS NOT NULL`

* **`ORDER BY`**
    * **역할**: 조회 결과를 특정 필드를 기준으로 정렬한다.
    * **예시**: `ORDER BY m.age DESC, m.username ASC`

---

### **2. 조인(Join)**

* **`JOIN` (또는 `INNER JOIN`)**
    * **역할**: 두 엔티티 간에 연관관계가 있는 데이터만 조회하는 내부 조인을 수행한다.
    * **예시**: `FROM Member m JOIN m.team t`

* **`LEFT JOIN` (또는 `LEFT OUTER JOIN`)**
    * **역할**: 왼쪽 엔티티를 기준으로, 연관된 오른쪽 엔티티가 없더라도 왼쪽 엔티티를 모두 조회하는 외부 조인을 수행한다.
    * **예시**: `FROM Member m LEFT JOIN m.team t`

* **`ON`**
    * **역할**: (JPA 2.1+) 조인 조건을 직접 지정할 때 사용한다. 연관관계 없는 엔티티를 조인하거나, 기존 조인에 조건을 추가할 수 있다.
    * **예시**: `LEFT JOIN Team t ON m.username = t.name`

* **`FETCH`**
    * **역할**: `JOIN FETCH` 형태로 사용되며, N+1 문제를 해결하기 위해 연관된 엔티티나 컬렉션을 즉시 함께 로딩하는 페치 조인을 수행한다. JPQL에만 있는 특별한 기능이다.
    * **예시**: `JOIN FETCH m.team`

---

### **3. 그룹화 및 집계**

* **`GROUP BY`**
    * **역할**: 특정 필드를 기준으로 결과를 그룹화하여, 각 그룹에 대한 집계 함수를 적용한다.
    * **예시**: `GROUP BY t.name`

* **`HAVING`**
    * **역할**: `GROUP BY`로 그룹화된 결과에 대해 추가적인 필터링 조건을 적용한다. 집계 함수를 조건으로 사용할 수 있다.
    * **예시**: `HAVING COUNT(m) > 10`

* **집계 함수**
    * **역할**: 그룹화된 데이터의 통계를 계산한다.
    * **종류**: `COUNT(x)`, `SUM(x)`, `AVG(x)`, `MAX(x)`, `MIN(x)`

---

### **4. 고급 키워드 및 표현식**

* **`DISTINCT`**
    * **역할**: 조회 결과에서 중복된 로우를 제거한다. 페치 조인 시 데이터 뻥튀기를 해결하기 위해 애플리케이션 레벨의 중복 제거도 수행한다.
    * **예시**: `SELECT DISTINCT t FROM Team t`

* **`new`**
    * **역할**: 조회 결과를 `Object[]`가 아닌, 지정된 DTO 클래스의 인스턴스로 직접 생성하여 반환한다.
    * **예시**: `SELECT new com.example.MyDto(m.username, m.age) FROM Member m`

* **`TYPE`**
    * **역할**: 상속 관계 매핑에서, 특정 타입의 엔티티만 필터링할 때 사용한다.
    * **예시**: `WHERE TYPE(i) = Book`

* **`TREAT`**
    * **역할**: (JPA 2.1+) 상속 관계 매핑에서, 부모 타입을 자식 타입으로 다운캐스팅하여 자식의 고유 필드에 접근할 때 사용한다.
    * **예시**: `WHERE TREAT(i AS Book).author = 'Kim'`

* **서브쿼리 연산자**
    * **역할**: `WHERE`, `HAVING` 절에서 서브쿼리의 결과와 비교할 때 사용한다.
    * **종류**: `[NOT] IN`, `[NOT] EXISTS`, `ALL`, `ANY`, `SOME`

---

### **5. 주요 내장 함수**

* **`CONCAT(str1, str2)`**: 문자열을 합친다.
* **`SUBSTRING(str, start, length)`**: 문자열을 자른다.
* **`TRIM()`, `LOWER()`, `UPPER()`**: 공백 제거 및 대소문자 변환.
* **`SIZE(collection)`**: 컬렉션의 크기를 반환한다. (`COUNT`와 유사한 SQL로 변환됨)
* **`COALESCE(val1, val2)`**: `val1`이 `NULL`이 아니면 `val1`을, `NULL`이면 `val2`를 반환한다.
* **`NULLIF(val1, val2)`**: `val1`과 `val2`가 같으면 `NULL`을, 다르면 `val1`을 반환한다.
* **`function('name', ...)`**: 데이터베이스 방언(Dialect)에 등록된 사용자 정의 함수나 DB 고유 함수를 호출한다.