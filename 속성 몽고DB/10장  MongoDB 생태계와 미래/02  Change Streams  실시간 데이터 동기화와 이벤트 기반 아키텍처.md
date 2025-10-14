## 02\. Change Streams: 실시간 데이터 동기화와 이벤트 기반 아키텍처

전통적인 애플리케이션 아키텍처에서, 데이터베이스의 변경 사항을 다른 시스템에 알리는 가장 흔한 방법은 \*\*폴링(Polling)\*\*이었습니다. 즉, 애플리케이션이 몇 초에 한 번씩 데이터베이스에 "새로운 데이터 있어?"라고 계속해서 물어보는 방식입니다. 이는 데이터베이스와 애플리케이션 양쪽에 불필요한 부하를 유발하며, 실시간이라고 부르기 어려운 지연이 발생합니다.

MongoDB **Change Streams**는 이러한 비효율적인 폴링 방식에 대한 현대적인 해답입니다. Change Streams는 데이터베이스의 컬렉션에서 발생하는 변경(insert, update, delete 등)을 \*\*실시간으로 구독(subscribe)\*\*하고, 변경 사항이 발생할 때마다 **푸시(push) 알림**처럼 즉시 받아볼 수 있게 해주는 강력한 기능입니다.

-----

### Change Streams의 작동 원리: Oplog를 재활용하다

Change Streams의 영리함은 완전히 새로운 기술을 발명한 것이 아니라, 7장에서 배운 레플리카 셋의 복제 메커니즘의 핵심인 **Oplog**를 활용한다는 점에 있습니다.

Oplog는 데이터베이스의 모든 쓰기 작업을 시간 순서대로 기록하는 내구성 있는 로그입니다. Change Streams는 바로 이 Oplog를 애플리케이션이 안전하게 '엿볼(tail)' 수 있도록 공식적인 API를 제공하는 것입니다. 애플리케이션은 `db.collection.watch()` 와 같은 메소드를 사용하여 특정 컬렉션에 대한 Change Stream을 열고, 새로운 이벤트가 발생할 때까지 대기하는 특별한 커서(cursor)를 유지합니다.

**핵심 기능: 재시작 가능성 (Resumability)**
만약 애플리케이션이 네트워크 문제로 잠시 연결이 끊어지더라도, Change Streams는 이벤트 유실 없이 안전하게 다시 시작할 수 있습니다. 모든 변경 이벤트에는 \*\*재시작 토큰(resume token)\*\*이 포함되어 있어, 애플리케이션은 마지막으로 수신한 이벤트의 토큰만 기억하고 있으면 됩니다. 재연결 시 이 토큰을 사용하여 정확히 그 지점부터 스트림을 이어갈 수 있습니다.

### 변경 이벤트(Change Event) 도큐먼트

Change Stream을 통해 전달되는 데이터는 다음과 같은 구조를 가집니다.

```json
{
  "_id": { <Resume Token> },
  "operationType": "update",
  "clusterTime": Timestamp(...),
  "ns": { "db": "ecommerce", "coll": "orders" },
  "documentKey": { "_id": "ORD001" },
  "updateDescription": {
    "updatedFields": { "status": "SHIPPED", "shippedAt": ISODate("...") },
    "removedFields": []
  },
  "fullDocument": { // fullDocument 옵션 사용 시 포함됨
    "_id": "ORD001",
    "status": "SHIPPED",
    "customerId": "user001",
    ...
  }
}
```

  * `operationType`: `insert`, `update`, `delete` 등 변경의 종류
  * `ns`: 변경이 발생한 데이터베이스와 컬렉션
  * `documentKey`: 변경된 도큐먼트의 `_id`
  * `updateDescription`: `update`의 경우, 어떤 필드가 어떻게 변경되었는지에 대한 상세 정보
  * `fullDocument`: (선택 사항) 변경 후의 도큐먼트 전체 모습. 이 옵션을 사용하면 변경된 데이터를 가져오기 위해 2차 쿼리를 날릴 필요가 없어 매우 유용합니다.

-----

### 주요 활용 시나리오: 이벤트 기반 아키텍처 구축

Change Streams는 마이크로서비스들이 서로의 상태를 직접 묻는 대신, 데이터베이스에서 발생하는 '이벤트'에 반응하여 독립적으로 동작하는 \*\*이벤트 기반 아키텍처(Event-Driven Architecture)\*\*를 구축하는 핵심 요소입니다.

1.  **실시간 동기화:** MongoDB의 데이터를 Elasticsearch(검색), Redis(캐시), 데이터 웨어하우스 등 다른 시스템으로 실시간 동기화할 수 있습니다. 하루에 한 번 실행되는 배치 작업 대신, 데이터가 변경되는 즉시 다른 시스템에 반영됩니다.

2.  **푸시 알림:** 채팅 애플리케이션에서 `messages` 컬렉션에 `insert` 이벤트가 발생하면, 알림 서비스가 이를 즉시 감지하여 상대방의 모바일 기기에 푸시 알림을 보낼 수 있습니다.

3.  **비즈니스 로직 트리거:** `orders` 컬렉션에 `status`가 `"PAID"`로 변경되는 `update` 이벤트가 발생하면, 배송 서비스가 이를 감지하여 자동으로 배송 준비 프로세스를 시작할 수 있습니다.

4.  **실시간 대시보드:** 관리자 대시보드에서 `orders` 컬렉션의 `insert` 이벤트를 구독하여, 새로운 주문이 들어올 때마다 화면을 새로고침하지 않아도 그래프와 통계가 실시간으로 업데이트되게 만들 수 있습니다.

#### Kotlin 코드 예제

Kotlin Coroutine 드라이버를 사용하여 `orders` 컬렉션의 변경 사항을 구독하는 예제입니다.

```kotlin
import kotlinx.coroutines.flow.collect
import kotlinx.coroutines.launch
import com.mongodb.client.model.changestream.FullDocument

// ... collection은 "orders"에 대한 MongoCollection 인스턴스
// ... Coroutine Scope 내에서 실행

// 1. Change Stream을 watch() 메소드로 엽니다.
val changeStreamFlow = collection.watch()
    .fullDocument(FullDocument.UPDATE_LOOKUP) // update 시 전체 문서를 포함시킵니다.

// 2. 별도의 코루틴에서 이벤트를 계속해서 수신합니다.
launch {
    changeStreamFlow.collect { changeEvent ->
        println("변경 이벤트 수신: ${changeEvent.operationType}")
        
        when (changeEvent.operationType) {
            OperationType.INSERT -> {
                val newOrder = changeEvent.fullDocument
                println("새 주문 접수: ${newOrder?.getObjectId("_id")}")
                // TODO: 주문 접수 알림 로직
            }
            OperationType.UPDATE -> {
                val updatedFields = changeEvent.updateDescription?.updatedFields
                println("주문 상태 변경: ${updatedFields}")
                // TODO: 배송 시작 로직
            }
            else -> {
                // 다른 타입의 이벤트 처리
            }
        }
    }
}
```

Change Streams는 애플리케이션의 결합도(coupling)를 낮추고, 시스템이 데이터 변경에 즉각적으로 반응하는 유연하고 확장 가능한 아키텍처를 구축할 수 있게 해주는 MongoDB의 핵심 기능입니다.