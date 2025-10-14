## 이종(Polyglot) 데이터베이스 간의 데이터 동기화 전략 (Kafka Connect, Debezium)

11장 02절에서 우리는 '최종 일관성'을 수용하기로 결정했습니다. 이제 남은 질문은 "어떻게?"입니다. `product-service`의 \*\*쓰기 DB(PostgreSQL)\*\*에서 발생한 변경 사항을, 어떻게 \*\*읽기 DB(Elasticsearch)\*\*에 안정적이고 효율적으로 복제(Replicate)할 수 있을까요?

가장 순진한 방법은 `ProductService`의 비즈니스 로직 코드 안에 Elasticsearch 클라이언트 코드를 직접 넣는 것입니다.

```kotlin
// (나쁜 예시) 애플리케이션이 직접 이중 쓰기를 수행
@Service
@Transactional
class ProductService(
    private val productRepository: ProductRepository, // RDBMS
    private val elasticsearchClient: ElasticsearchClient // Elasticsearch
) {
    fun updateProductPrice(command: UpdatePriceCommand) {
        // 1. RDBMS에 저장
        val product = productRepository.findById(...)
        product.changePrice(...)
        productRepository.save(product)
        
        // 2. Elasticsearch에도 저장
        elasticsearchClient.index(...) 
    }
}
```

이 방식은 09장의 '듀얼 라이트' 문제와 정확히 동일한 이유로 **최악의 안티패턴**입니다. 비즈니스 로직이 데이터 동기화라는 인프라 문제에 오염되고, 두 DB 중 하나라도 실패하면 데이터 정합성이 깨집니다.

-----

### CDC (Change Data Capture): "DB의 변경 사항을 낚아채라"

이 문제를 해결하는 가장 현대적이고 견고한 패턴이 바로 \*\*CDC (변경 데이터 캡처)\*\*입니다.

**CDC**는 애플리케이션 코드를 전혀 건드리지 않고, **데이터베이스 자체에서 발생하는 모든 변경 내역(INSERT, UPDATE, DELETE)을 이벤트 스트림으로 캡처**하는 기술입니다. 이는 데이터베이스의 **트랜잭션 로그(Transaction Log)** 또는 \*\*바이너리 로그(Binary Log)\*\*를 실시간으로 읽어오는 방식으로 동작합니다.

  * **장점:**
      * **비침투적(Non-invasive):** 기존 애플리케이션 코드(`ProductService`)는 **단 한 줄도 수정할 필요가 없습니다.** 데이터 동기화의 책임이 애플리케이션에서 완전히 분리됩니다.
      * **신뢰성:** DB의 트랜잭션 로그는 데이터 변경이 커밋된 후에야 기록되므로, CDC는 **실제로 DB에 반영된 변경 사항만**을 이벤트로 발행하는 것을 보장합니다. 절대 데이터가 유실되지 않습니다.
      * **낮은 부하:** 애플리케이션이 DB를 폴링(Polling)하는 방식이 아니라, DB 로그를 직접 읽으므로 DB에 주는 부하가 매우 적습니다.

-----

### Kafka Connect와 Debezium: CDC를 위한 드림팀

이 CDC를 Kafka 생태계에서 구현하기 위한 최고의 조합이 바로 **Kafka Connect**와 **Debezium**입니다.

1.  **Kafka Connect:**

      * Kafka와 다른 시스템(DB, S3, Elasticsearch 등) 간의 **데이터 파이프라인을 구축하고 실행하기 위한 프레임워크**입니다.
      * Kafka Connect는 \*\*소스 커넥터(Source Connector)\*\*와 \*\*싱크 커넥터(Sink Connector)\*\*라는 두 종류의 '플러그인'을 실행하는 컨테이너 역할을 합니다.
          * **소스 커넥터:** 외부 시스템에서 데이터를 **가져와(Source)** Kafka 토픽으로 보내는 역할.
          * **싱크 커넥터:** Kafka 토픽의 데이터를 **가져가(Sink)** 외부 시스템에 쓰는 역할.

2.  **Debezium (디비지움):**

      * \*\*가장 강력하고 널리 쓰이는 CDC '소스 커넥터'\*\*입니다. PostgreSQL, MySQL, MongoDB 등 다양한 DB의 트랜잭션 로그를 실시간으로 읽어, 변경 내역을 구조화된 이벤트로 변환한 뒤 Kafka 토픽으로 전송합니다.

-----

### `product-service`의 CQRS 동기화 파이프라인 구축

이제 이 조합을 사용하여 `product-service`의 쓰기 DB(PostgreSQL)와 읽기 DB(Elasticsearch)를 동기화하는 완전 자동화된 파이프라인을 설계해 봅시다.

1.  **[DB 변경 발생]** 개발자가 `Product` 엔티티의 가격을 변경하고 `productRepository.save(product)`를 호출합니다. PostgreSQL의 `products` 테이블에 `UPDATE` 쿼리가 실행되고, 이 변경 내역이 트랜잭션 로그(WAL, Write-Ahead Log)에 기록됩니다.

2.  **[Debezium 소스 커넥터]** Kafka Connect 위에서 실행 중인 **Debezium PostgreSQL 커넥터**는 이 트랜잭션 로그의 변경 사항을 실시간으로 감지합니다.

3.  **[Kafka로 이벤트 발행]** Debezium은 `UPDATE` 내역을 `ProductPriceChanged`와 같은 의미 있는 이벤트로 변환하여, `products`라는 Kafka 토픽으로 보냅니다. 이벤트에는 변경 전(`before`) 데이터와 변경 후(`after`) 데이터가 모두 포함됩니다.

4.  **[Elasticsearch 싱크 커넥터]** Kafka Connect 위에서 실행 중인 **Elasticsearch 싱크 커넥터**는 `products` 토픽을 구독하고 있습니다.

5.  **[Elasticsearch 인덱싱]** 싱크 커넥터는 새로운 이벤트를 받아, 이벤트의 `after` 데이터를 사용하여 Elasticsearch의 `products` 인덱스에 있는 해당 상품 도큐먼트를 \*\*갱신(Update)\*\*합니다.

이 모든 과정은 `product-service`의 **애플리케이션 코드와는 100% 무관하게**, 백그라운드에서 완전 자동으로 이루어집니다. `product-service`는 그저 자신의 RDBMS에 데이터를 쓰고 읽을 뿐, 자신의 데이터가 Elasticsearch에 복제되고 있다는 사실조차 알 필요가 없습니다.

이것이 바로 `Database per Service` 패턴과 폴리글랏 퍼시스턴스가 야기하는 '데이터 동기화' 문제에 대한 가장 현대적이고 견고한 해답입니다. 다음 절에서는 이 파이프라인의 마지막 조각, 즉 `DB 트랜잭션`과 `이벤트 발행`의 원자성을 보장하는 **'트랜잭셔널 아웃박스'** 패턴에 대해 알아보겠습니다.