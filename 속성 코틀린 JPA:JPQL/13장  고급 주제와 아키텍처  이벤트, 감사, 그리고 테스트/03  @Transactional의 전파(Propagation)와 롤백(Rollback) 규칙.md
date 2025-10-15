## 03\. `@Transactional`의 전파(Propagation)와 롤백(Rollback) 규칙

우리는 지금까지 `@Transactional` 어노테이션을 마치 트랜잭션을 시작하고 끝내는 '스위치'처럼 단순하게 사용해왔다. 하지만 이 작은 어노테이션 뒤에는 여러 트랜잭션이 서로 상호작용하는 방식을 제어하는 정교하고 중요한 규칙들이 숨어있다. 바로 \*\*전파(Propagation)\*\*와 **롤백(Rollback)** 규칙이다. 이들을 제대로 이해하지 못하면, 우리는 데이터 정합성이 깨지는 심각한 버그를 만들고도 그 원인을 찾지 못해 헤매게 될 것이다.

-----

### **트랜잭션 전파 (Propagation): 트랜잭션의 흐름 제어**

만약 트랜잭션이 이미 진행 중인 메서드 안에서, `@Transactional`이 붙은 다른 메서드를 호출하면 어떻게 될까? 새로운 트랜잭션을 시작해야 할까? 아니면 기존 트랜잭션에 참여해야 할까? 이처럼 **트랜잭션의 경계에서 다른 트랜잭션을 만났을 때 어떻게 행동할지를 결정하는 규칙**이 바로 **트랜잭션 전파**다.

스프링은 `@Transactional`의 `propagation` 속성을 통해 다양한 전파 레벨을 제공하며, 가장 중요한 두 가지는 다음과 같다.

  * **`REQUIRED` (기본값)**

      * **설명**: 현재 진행 중인 트랜잭션이 있으면 그 트랜잭션에 참여하고, 없으면 새로운 트랜잭션을 시작한다.
      * **동작**: 하나의 거대한 물리적 트랜잭션 안에서 모든 로직이 실행된다. 내부 메서드에서 예외가 발생하면, 그 예외는 외부 메서드까지 전파되어 **전체 트랜잭션이 롤백**된다. 대부분의 경우에 사용하는 가장 일반적이고 자연스러운 전파 레벨이다.

    <!-- end list -->

    ```kotlin
    class OuterService {
        @Transactional // (1) 여기서 새로운 트랜잭션 시작
        fun outer() {
            innerService.inner()
        }
    }
    class InnerService {
        @Transactional // (2) 이미 시작된 트랜잭션이 있으므로 여기에 참여
        fun inner() {
            // ... 로직 수행
            throw RuntimeException() // (3) 예외 발생 시 outer()까지 롤백됨
        }
    }
    ```

  * **`REQUIRES_NEW`**

      * **설명**: **항상 새로운 트랜잭션을 시작한다.** 이미 진행 중인 트랜잭션이 있더라도, 그 트랜잭션은 잠시 보류(suspend)시키고, 완전히 독립적인 새로운 트랜잭션을 생성하여 실행한다.
      * **동작**: 내부 메서드는 외부 트랜잭션과 별개의 물리적 트랜잭션을 가진다. 따라서 내부 트랜잭션에서 발생한 예외로 **내부 트랜잭션만 롤백되고, 외부 트랜잭션에는 영향을 주지 않을 수 있다.** (물론 예외를 `catch` 하지 않으면 외부까지 전파되어 결국 전체 롤백된다.) 주로 하나의 로직 안에서 특정 작업(예: 로그 기록)은 반드시 성공적으로 커밋되어야 할 때 사용된다.

    <!-- end list -->

    ```kotlin
    class OrderService {
        @Transactional // (1) 주문 트랜잭션 시작
        fun placeOrder() {
            // ... 주문 로직 ...
            try {
                logHistoryService.log() // (2) 새로운 트랜잭션 시작
            } catch (e: Exception) {
                // 로그 기록이 실패해도 주문은 계속 진행
            }
        }
    }
    class LogHistoryService {
        @Transactional(propagation = Propagation.REQUIRES_NEW)
        fun log() {
            // ... 로그 DB에 기록 ...
            // 여기서 문제가 생겨도 placeOrder() 트랜잭션은 롤백되지 않는다.
        }
    }
    ```

-----

### **트랜잭션 롤백 규칙**

스프링의 `@Transactional`은 예외가 발생했을 때 트랜잭션을 자동으로 롤백해주는 편리한 기능을 제공한다. 하지만 여기서 매우 중요한 규칙이 있다.

> **스프링은 기본적으로 `RuntimeException`(Unchecked Exception)과 `Error`가 발생했을 때만 트랜잭션을 롤백한다.**

`IOException`이나 비즈니스적으로 의미가 있는 커스텀 예외처럼 **`Exception`(Checked Exception)을 상속받은 예외가 발생하면, 스프링은 이를 복구 가능한 예외로 간주하고 트랜잭션을 롤백하지 않고 커밋해버린다\!** 이는 많은 개발자들이 실수하는 지점으로, 데이터 정합성을 깨뜨리는 심각한 버그의 원인이 될 수 있다.

이 기본 동작을 변경하려면 `@Transactional`의 `rollbackFor`와 `noRollbackFor` 속성을 사용하면 된다.

  * **`rollbackFor`**: 특정 예외(또는 그 자식 예외)가 발생했을 때 강제로 롤백하도록 지정한다.

    ```kotlin
    // Checked Exception인 MyBusinessException이 발생해도 롤백하도록 설정
    @Transactional(rollbackFor = [MyBusinessException::class])
    fun process() {
        // ...
        throw MyBusinessException("Something went wrong but should rollback")
    }
    ```

  * **`noRollbackFor`**: 기본적으로 롤백 대상인 예외(예: `RuntimeException`)가 발생했음에도 불구하고 롤백하지 않고 커밋하도록 지정한다.

    ```kotlin
    // 특정 RuntimeException은 롤백하지 않도록 설정
    @Transactional(noRollbackFor = [MyMinorRuntimeException::class])
    fun handleMinorError() {
        // ...
        throw MyMinorRuntimeException("It's okay to commit")
    }
    ```

`@Transactional`의 전파와 롤백 규칙을 이해하는 것은 단순히 기능을 아는 것을 넘어, 여러 서비스가 복잡하게 얽힌 환경에서 데이터의 흐름을 정확하게 제어하고 안정성을 확보하는 아키텍처 설계의 핵심이다.

이제 JPA를 DDD의 관점에서 어떻게 바라보고 설계해야 하는지, 다음 절에서 알아보자.