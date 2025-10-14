## 03\. Time Series 컬렉션의 구조와 시계열 데이터 처리

\*\*시계열 데이터(Time-series data)\*\*는 IoT 센서 판독값, 주식 시장 가격, 시스템 모니터링 메트릭 등 시간의 흐름에 따라 순차적으로 기록되는 모든 데이터를 의미합니다. 이러한 데이터는 초당 수백만 건씩 수집될 수 있는 엄청난 양, 그리고 시간 범위에 기반한 조회 및 집계가 주를 이룬다는 뚜렷한 특징을 가집니다.

과거에는 이러한 대규모 시계열 데이터를 일반적인 데이터베이스에 저장하는 것이 비효율적이었습니다. 측정값 하나당 도큐먼트 하나를 저장하는 방식은 엄청난 스토리지 오버헤드와 느린 쿼리 성능을 유발했기 때문입니다. MongoDB 5.0부터 도입된 **Time Series 컬렉션**은 이러한 문제를 해결하기 위해 특별히 설계된, 시계열 데이터 처리에 고도로 최적화된 컬렉션 유형입니다.

-----

### Time Series 컬렉션의 내부 구조: 스마트한 버킷팅(Bucketing)

Time Series 컬렉션의 마법은, 개발자는 일반 컬렉션처럼 개별 측정값 도큐먼트를 `insertOne`하기만 하면 되지만, MongoDB가 **내부적으로는 완전히 다른 방식으로 데이터를 저장**한다는 점에 있습니다.

MongoDB는 디스크에 측정값 하나하나를 별도의 도큐먼트로 저장하는 대신, 시간적으로 서로 근접한 여러 측정값들을 **하나의 거대한 '버킷(bucket)' 도큐먼트**로 자동으로 묶어서 저장합니다.

**비유:** 매일 발생하는 수백 건의 영수증(측정값)을 하나씩 개별 파일에 보관하는 대신, 회계 담당자가 한 시간 동안 발생한 영수증을 모두 모아 스테이플러로 묶어('버킷팅'), '오전 10시 영수증 묶음'이라는 단 하나의 파일(버킷 도큐먼트)로 보관하는 것과 같습니다. 파일의 총개수가 획기적으로 줄어들어 관리와 검색이 훨씬 효율적이 됩니다.

이 내부 버킷 도큐먼트는 데이터를 고도로 압축된 컬럼(columnar) 형식으로 저장하여, 저장 공간 효율을 극대화하고 시간 범위 쿼리 시 디스크 I/O를 최소화합니다.

-----

### Time Series 컬렉션 생성 및 사용 (Kotlin 기준)

Time Series 컬렉션을 생성할 때는 최소 두 가지 옵션을 반드시 지정해야 합니다.

  * `timeField`: 측정 시간이 기록된 필드의 이름
  * `metaField`: 데이터 소스를 식별하는 메타데이터가 담긴 필드의 이름 (예: 센서 ID, 주식 종목 코드)

<!-- end list -->

```kotlin
import com.mongodb.client.model.CreateCollectionOptions
import com.mongodb.client.model.TimeSeriesGranularity
import com.mongodb.client.model.TimeSeriesOptions

// ... database는 MongoDatabase 인스턴스라고 가정
// Coroutine Scope 내에서 실행

// 1. Time Series 컬렉션 옵션 정의
val timeSeriesOptions = TimeSeriesOptions("timestamp")       // 시간 필드 지정
    .metaField("metadata.sensorId")                      // 메타 필드 지정
    .granularity(TimeSeriesGranularity.SECONDS)          // 데이터의 시간 간격 힌트 (선택 사항)

val createCollectionOptions = CreateCollectionOptions()
    .timeSeriesOptions(timeSeriesOptions)

// 2. 컬렉션 생성
database.createCollection("weather", createCollectionOptions)
```

**데이터 삽입:** 데이터 삽입은 일반 컬렉션과 완전히 동일합니다.

```kotlin
val weatherCollection = database.getCollection<Document>("weather")

// 여러 측정값을 한 번에 삽입하는 것이 훨씬 효율적입니다.
val measurements = listOf(
    Document("metadata", Document("sensorId", "A001").append("location", "서울"))
        .append("timestamp", Instant.now())
        .append("temperature", 22.1),
    Document("metadata", Document("sensorId", "A001").append("location", "서울"))
        .append("timestamp", Instant.now().plusSeconds(5))
        .append("temperature", 22.2)
)
weatherCollection.insertMany(measurements)
```

-----

### Time Series 컬렉션의 압도적인 장점

1.  **획기적인 저장 공간 효율성:** 버킷팅과 컬럼 형식 압축을 통해, 일반 컬렉션 대비 스토리지 사용량을 최대 80\~90%까지 절감할 수 있습니다.
2.  **높은 쓰기 처리량:** 시계열 데이터의 주된 패턴인 대량의 순차적 쓰기(ingest)에 최적화되어 있습니다.
3.  **빠른 쿼리 성능:**
      * **I/O 감소:** "지난 1시간 동안의 모든 데이터"를 조회할 때, 수천 개의 개별 도큐먼트 대신 몇 개의 버킷 도큐먼트만 읽으면 되므로 디스크 I/O가 크게 줄어듭니다.
      * **클러스터형 인덱스 (Clustered Index):** Time Series 컬렉션은 시간(`timeField`)을 기준으로 데이터가 디스크에 **물리적으로 정렬되어 저장**됩니다. 이는 시간 범위 쿼리 시, 인덱스를 보고 다시 데이터 파일을 찾아가는 비효율 없이, 디스크의 연속된 공간만 순차적으로 읽으면 되므로 엄청난 성능 향상을 가져옵니다. **데이터가 곧 인덱스인 셈입니다.**
4.  **애그리게이션 최적화:** `＄group`과 함께 시간 버킷팅을 수행하는 `$dateTrunc`와 같은 새로운 애그리게이션 연산자를 통해 시계열 데이터 분석을 더욱 빠르고 쉽게 수행할 수 있습니다.

IoT, 금융, 로그 분석, 모니터링 등 대규모 시계열 데이터를 다루는 모든 워크로드에서 Time Series 컬렉션은 이제 선택이 아닌 필수입니다. 이는 MongoDB가 단순한 문서 데이터베이스를 넘어, 특정 도메인의 문제를 해결하는 전문적인 데이터 플랫폼으로 진화하고 있음을 보여주는 대표적인 사례입니다.