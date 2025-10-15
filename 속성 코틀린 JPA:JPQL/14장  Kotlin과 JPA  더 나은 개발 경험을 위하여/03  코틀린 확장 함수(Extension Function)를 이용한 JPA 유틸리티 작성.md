## 03\. 코틀린 확장 함수(Extension Function)를 이용한 JPA 유틸리티 작성

코틀린의 가장 강력하고 매력적인 기능 중 하나는 \*\*확장 함수(Extension Function)\*\*다. 확장 함수는 우리가 직접 수정할 수 없는 기존 클래스(예: 외부 라이브러리의 클래스)에 마치 원래부터 있었던 것처럼 새로운 메서드를 추가할 수 있게 해주는 기능이다.

스프링 데이터 JPA를 사용하다 보면 `Repository`의 `findById()` 메서드가 반환하는 `Optional<T>` 타입을 자주 다루게 된다. `Optional`은 `null`을 더 안전하게 다루기 위한 훌륭한 래퍼(wrapper) 클래스지만, 값이 없을 때 예외를 던지는 로직은 매번 반복적으로 작성해야 한다.

```kotlin
// 반복되는 보일러플레이트 코드
val member = memberRepository.findById(id)
    .orElseThrow { EntityNotFoundException("Member not found with id: $id") }
```

이런 반복적인 코드를 코틀린의 확장 함수를 사용하면 매우 우아하게 제거할 수 있다.

-----

### **`Optional`을 위한 확장 함수 만들기**

우리는 `Optional<T>` 클래스에 `orElseThrow`의 커스텀 버전을 확장 함수로 추가해 볼 것이다.

**RepositoryExtensions.kt**

```kotlin
// 최상위 레벨에 함수를 선언하여 유틸리티로 사용한다.
package com.masterclass.jpamaster.repository

import jakarta.persistence.EntityNotFoundException
import java.util.Optional

// Optional<T> 클래스에 orElseThrow 커스텀 버전을 추가한다.
fun <T> Optional<T>.orElseThrow(id: Long, entityName: String = "Entity"): T {
    return this.orElseThrow {
        EntityNotFoundException("$entityName not found with id: $id")
    }
}
```

  * `fun <T> Optional<T>.orElseThrow(...)`: `Optional<T>` 클래스에 `orElseThrow`라는 이름의 새로운 함수를 추가하겠다는 선언이다.
  * `this`: 확장 함수 내부에서 `this`는 수신 객체, 즉 `Optional` 인스턴스 그 자체를 가리킨다.

### **확장 함수를 사용하여 코드 개선하기**

이제 우리가 만든 확장 함수를 사용하여 기존의 서비스 코드를 리팩토링해 보자.

**기존 코드**

```kotlin
@Transactional
fun findMember(id: Long): Member {
    return memberRepository.findById(id)
        .orElseThrow { EntityNotFoundException("Member not found with id: $id") }
}
```

**개선된 코드**

```kotlin
// import com.masterclass.jpamaster.repository.orElseThrow // 확장 함수를 import 한다.

@Transactional
fun findMember(id: Long): Member {
    // 훨씬 더 간결하고 의도가 명확해졌다.
    return memberRepository.findById(id).orElseThrow(id, "Member")
}
```

코드가 훨씬 간결해지고 가독성이 높아졌다. `orElseThrow(id, "Member")`라는 호출만 봐도 "ID를 기반으로 엔티티를 찾다가 없으면 예외를 던지는구나"라는 의도가 명확하게 드러난다.

-----

확장 함수는 이처럼 `Optional` 처리뿐만 아니라, `Page<T>`를 특정 `DTO`의 `Page`로 변환하는 로직, `Specification`을 조합하는 로직 등 JPA를 사용하면서 발생하는 거의 모든 종류의 반복적인 유틸리티성 코드를 캡슐화하고 재사용하는 데 매우 강력한 힘을 발휘한다.

이는 단순히 코드를 줄이는 것을 넘어, 우리의 데이터 접근 코드를 더 읽기 쉽고, 유지보수하기 좋으며, 도메인에 더 집중된 \*\*'우리만의 DSL(Domain Specific Language)'\*\*처럼 만들어주는 효과를 가져온다.

이제 코틀린의 동시성 모델인 코루틴이 JPA와 만났을 때 어떤 일이 벌어지는지, 다음 절에서 그 한계와 해결책을 알아보자.