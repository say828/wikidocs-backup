## 05\. 빅테크의 샤딩 전략: 존 샤딩(Zone Sharding)과 태그 어웨어 샤딩(Tag-Aware Sharding)

지금까지 우리는 자동화된 밸런서가 데이터를 클러스터 전반에 '균등하게' 분산시키는 것을 목표로 한다고 배웠습니다. 하지만 페이스북, 구글, 넷플릭스와 같은 글로벌 빅테크 기업들의 아키텍처를 들여다보면, 때로는 '균등함'보다 \*\*'의도된 불균형'\*\*이 더 중요한 비즈니스 요구사항일 때가 많습니다.

예를 들어, 유럽 사용자의 데이터는 GDPR(개인정보보호규정) 준수를 위해 반드시 유럽 대륙 내의 데이터센터에만 저장되어야 하거나, 자주 접근하는 '핫(Hot)' 데이터는 고성능 SSD 샤드에, 오래된 '콜드(Cold)' 데이터는 저비용 HDD 샤드에 저장하여 인프라 비용을 최적화해야 하는 경우입니다.

이러한 고급 데이터 배치 요구사항을 해결하기 위해 MongoDB가 제공하는 기능이 바로 **존 샤딩(Zone Sharding)**, 또는 \*\*태그 어웨어 샤딩(Tag-Aware Sharding)\*\*입니다. 이는 밸런서의 자동화된 균형 조정 능력에 더하여, 관리자가 데이터의 물리적 위치를 직접 제어할 수 있는 강력한 기능입니다.

-----

### 존 샤딩의 개념

존 샤딩은 샤드(Shard) 그룹에 특정 \*\*태그(Tag)\*\*를 부여하고, 샤드 키 값의 특정 \*\*범위(Range)\*\*를 해당 태그와 연결하는 방식으로 동작합니다.

**글로벌 물류 창고 비유:**

  * **일반 샤딩:** 전 세계에서 들어오는 모든 택배를 비어있는 창고에 무작위로 균등하게 쌓습니다.
  * **존 샤딩:** 독일 프랑크푸르트 창고에는 "EUROPE-ZONE"이라는 태그를, 미국 버지니아 창고에는 "US-ZONE"이라는 태그를 붙입니다. 그리고 "발송지가 유럽인 모든 택배는 반드시 EUROPE-ZONE 태그가 붙은 창고에만 보관해야 한다"라는 규칙을 만듭니다.

이 규칙이 설정되면, 밸런서는 더 이상 청크를 무작위로 분산시키지 않습니다. 밸런서의 최우선 임무는 **존(Zone) 정책을 준수하는 것**이 됩니다. 즉, 유럽에서 온 택배(청크)가 미국 창고에 잘못 보관되어 있다면, 밸런서는 이를 감지하고 즉시 유럽 창고로 옮기는 작업을 수행합니다. 존 내부에서는 여전히 청크를 균등하게 분산시키려고 노력합니다.

-----

### 주요 활용 시나리오

#### 시나리오 1: 데이터의 지리적 분산 (Data Locality & GDPR Compliance)

**문제:** 전 세계 사용자를 대상으로 하는 소셜 미디어 애플리케이션이 있습니다. 유럽 사용자의 데이터는 GDPR 규정에 따라 반드시 EU 내 데이터센터에 저장되어야 하며, 동시에 유럽 사용자들이 가장 빠른 응답 속도를 경험하게 하고 싶습니다.

**해결책:**

1.  **샤드 배치 및 태그 지정:**

      * `shard-EU` (레플리카 셋)는 AWS 프랑크푸르트(eu-central-1) 리전에 배포하고 "EU" 태그를 할당합니다.
      * `shard-US` (레플리카 셋)는 AWS 버지니아(us-east-1) 리전에 배포하고 "US" 태그를 할당합니다.

```kotlin
// 이 명령어들은 admin 데이터베이스에 대해 실행되어야 합니다.
// client는 MongoClient 인스턴스, Coroutine Scope 내에서 실행
val adminDb = client.getDatabase("admin")

adminDb.runCommand(Document("addShardToZone", "shard-EU").append("zone", "EU_ZONE"))
adminDb.runCommand(Document("addShardToZone", "shard-US").append("zone", "US_ZONE"))
```

2.  **존 범위 정의:** `users` 컬렉션을 `{ user_country: 1, _id: 1 }` 과 같은 복합키로 샤딩하고, `user_country` 값에 따라 존 범위를 설정합니다.

```kotlin
// user_country가 "DE"(독일)인 문서는 EU_ZONE으로 지정
adminDb.runCommand(
    Document("updateZoneKeyRange", "app.users")
        .append("min", Document("user_country", "DE"))
        .append("max", Document("user_country", "DF")) // "DE" 와 같거나 크고, "DF" 보다 작은 범위
        .append("zone", "EU_ZONE")
)
// ... 다른 유럽 국가들에 대해서도 범위 추가 ...
```

**결과:** 이제 `user_country`가 "DE"인 신규 회원이 가입하면, 해당 도큐먼트는 밸런서에 의해 자동으로 프랑크푸르트에 위치한 `shard-EU`에만 저장됩니다. 유럽 사용자의 쿼리는 가장 가까운 데이터센터로 라우팅되어 지연 시간이 최소화되고, 데이터 주권 규정도 완벽하게 준수할 수 있습니다.

#### 시나리오 2: 데이터 티어링 (Hot/Cold Data Tiering)

**문제:** IoT 센서로부터 매초 수십만 건의 시계열 데이터가 수집됩니다. 최근 7일간의 '핫' 데이터는 실시간 대시보드에 사용되어 매우 빠른 조회가 필요하므로 고성능 NVMe SSD 샤드에 저장해야 합니다. 7일이 지난 '콜드' 데이터는 가끔 분석용으로만 사용되므로 저렴한 대용량 HDD 샤드에 보관하여 비용을 최적화하고 싶습니다.

**해결책:**

1.  **샤드 구성 및 태그 지정:**
      * `shard-hot`는 고사양 CPU와 NVMe SSD로 구성하고 "HOT" 태그를 부여합니다.
      * `shard-cold`는 일반 사양의 CPU와 대용량 HDD로 구성하고 "COLD" 태그를 부여합니다.
2.  **존 범위 정의:** 컬렉션을 `timestamp` 필드로 샤딩하고, 현재 시간을 기준으로 존 범위를 동적으로 관리하는 스크립트를 주기적으로 실행합니다.

```kotlin
import java.time.Instant
import java.time.temporal.ChronoUnit
import java.util.Date
import org.bson.types.MaxKey
import org.bson.types.MinKey

// ... adminDb는 admin 데이터베이스에 대한 참조 ...

val sevenDaysAgo = Instant.now().minus(7, ChronoUnit.DAYS)

// 최근 7일 데이터는 HOT 존으로 지정
adminDb.runCommand(
    Document("updateZoneKeyRange", "iot.sensor_data")
        .append("min", Document("timestamp", Date.from(sevenDaysAgo)))
        .append("max", Document("timestamp", MaxKey()))
        .append("zone", "HOT")
)

// 7일 이전 데이터는 COLD 존으로 지정
adminDb.runCommand(
    Document("updateZoneKeyRange", "iot.sensor_data")
        .append("min", Document("timestamp", MinKey()))
        .append("max", Document("timestamp", Date.from(sevenDaysAgo)))
        .append("zone", "COLD")
)
```

**결과:** 시간이 지남에 따라 데이터는 자연스럽게 '핫'에서 '콜드'로 변합니다. 밸런서는 매일 변경되는 존 범위를 인지하고, 어제까지 '핫'이었던 데이터 청크를 자동으로 `shard-hot`에서 `shard-cold`로 이동시킵니다. 이를 통해 관리자의 개입 없이 비용과 성능이 최적화된 데이터 티어링 아키텍처가 자동으로 운영됩니다.

존 샤딩은 단순한 부하 분산을 넘어, 데이터의 가치와 특성에 따라 물리적 위치를 제어하는 정교한 데이터 아키텍처 설계의 정수입니다. 이 기능을 통해 MongoDB는 진정한 의미의 글로벌 스케일 데이터 플랫폼을 구축하는 강력한 기반을 제공합니다.