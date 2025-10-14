## 01\. 리포지토리(Repository) 인터페이스의 마법: `JpaRepository` 심층 분석

`interface MemberRepository : JpaRepository<Member, Long>`

우리가 작성한 이 한 줄의 코드에는 스프링 데이터 JPA의 수많은 기능이 압축되어 있다. 이 마법의 비밀을 풀기 위해 `JpaRepository`라는 인터페이스의 계층 구조를 깊이 파고들어 보자. `JpaRepository`는 사실 여러 인터페이스를 상속받아 만들어진 최종 결과물이다.

**`JpaRepository`의 상속 구조**

```
JpaRepository<T, ID>
  └── extends PagingAndSortingRepository<T, ID>
        └── extends CrudRepository<T, ID>
              └── extends Repository<T, ID>
```

이 계층 구조를 거슬러 올라가며 각 인터페이스가 어떤 역할을 하는지 살펴보면, 스프링 데이터 JPA의 설계 사상을 엿볼 수 있다.

-----

### **`Repository<T, ID>`**

가장 최상위에 있는 인터페이스다. 놀랍게도 이 인터페이스는 아무런 메서드도 정의하고 있지 않은, 텅 빈 \*\*마커 인터페이스(Marker Interface)\*\*다. 이 인터페이스의 유일한 역할은 스프링에게 "이 인터페이스는 데이터 접근을 위한 리포지토리 빈으로 등록해 줘\!"라고 알려주는 것이다.

-----

### **`CrudRepository<T, ID>`**

이름에서 알 수 있듯이, 데이터베이스의 **CRUD(Create, Read, Update, Delete) 기능을 담당**하는 핵심적인 메서드들이 여기에 정의되어 있다.

  * **`save(S entity)`**: 새로운 엔티티는 저장하고, 이미 존재하는 엔티티는 병합(`merge`)한다.
  * **`findById(ID id)`**: ID를 기준으로 엔티티 하나를 조회한다. `Optional<T>`을 반환하여 Null-Safety를 보장한다.
  * **`findAll()`**: 모든 엔티티를 조회한다.
  * **`count()`**: 엔티티의 총개수를 반환한다.
  * **`delete(T entity)`**: 특정 엔티티를 삭제한다.
  * **`existsById(ID id)`**: 해당 ID를 가진 엔티티가 존재하는지 확인한다.

이 `CrudRepository`만 상속받아도 대부분의 기본적인 데이터 접근 기능은 해결된다.

-----

### **`PagingAndSortingRepository<T, ID>`**

`CrudRepository`의 기능을 모두 물려받으면서, 이름 그대로 **페이징(Paging)과 정렬(Sorting) 기능**을 추가로 제공한다.

  * **`findAll(Sort sort)`**: 특정 조건으로 정렬하여 모든 엔티티를 조회한다.
  * **`findAll(Pageable pageable)`**: 페이징 처리(페이지 번호, 페이지 크기)와 정렬을 함께 사용하여 특정 페이지의 데이터만 조회한다. `Page<T>`라는 특별한 객체를 반환하는데, 여기에는 조회된 데이터 목록뿐만 아니라 전체 페이지 수, 전체 데이터 개수 등 페이징 처리에 필요한 모든 정보가 담겨있다.

대용량 데이터를 다루는 현대적인 웹 애플리케이션에서 페이징과 정렬은 선택이 아닌 필수이므로, 이 인터페이스의 역할은 매우 중요하다.

-----

### **`JpaRepository<T, ID>`**

드디어 최종 목적지다. `JpaRepository`는 `PagingAndSortingRepository`의 모든 기능을 포함하면서, JPA 기술에 특화된 몇 가지 유용한 메서드를 추가로 제공한다.

  * **`findAll()`**: `CrudRepository`의 `findAll()`은 `Iterable<T>`을 반환하는 반면, `JpaRepository`는 더 사용하기 편한 `List<T>`를 반환하도록 오버라이딩했다.
  * **`flush()`**: 영속성 컨텍스트의 변경 내용을 데이터베이스에 강제로 동기화한다.
  * **`saveAndFlush(S entity)`**: 엔티티를 저장하는 즉시 플러시하여 DB에 반영한다.
  * **`deleteInBatch(Iterable<T> entities)`**: 여러 엔티티를 한 번의 `DELETE` 쿼리로 삭제하여 성능을 최적화한다.

결론적으로, 우리가 `JpaRepository`를 상속받는다는 것은 이 모든 상위 인터페이스의 기능을 한 번에 물려받아, 기본적인 CRUD부터 페이징, 정렬, 그리고 JPA 특화 기능까지 모두 사용할 수 있는 완전한 리포지토리를 단숨에 만들어내는 것과 같다.

하지만 이 제공된 메서드만으로는 부족하다. "특정 이름(`username`)으로 회원을 조회"하거나 "특정 나이보다 많은 회원을 조회"하는 것과 같은 비즈니스 요구사항을 처리하려면 어떻게 해야 할까? 다음 절에서 스프링 데이터 JPA의 가장 경이로운 기능인 '쿼리 메서드'에 대해 알아본다.