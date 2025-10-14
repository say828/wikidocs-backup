## Debezium과 Kafka Connect: CDC(Change Data Capture) 구축

우리는 11장 03절에서 \*\*CDC(Change Data Capture)\*\*를 '데이터베이스의 변경 사항을 이벤트 스트림으로 캡처하는 기술'이라고 소개했습니다. CDC는 서비스 간의 데이터 동기화, 트랜잭셔널 아웃박스 패턴 구현 등 현대 MSA에서 발생하는 수많은 데이터 문제를 해결하는 핵심적인 인프라입니다.

부록에서는 이 CDC 파이프라인의 핵심 컴포넌트인 **Kafka Connect**와 **Debezium**을 어떻게 설치하고 설정하는지 실용적인 가이드를 제공합니다.

-----

### \#\# Kafka Connect: "Kafka를 위한 플러그인 런처"

**Kafka Connect**는 Kafka와 다른 시스템(DB, S3 등) 간의 데이터 파이프라인을 실행하기 위한 **독립적인 분산 플랫폼**입니다. `Spring Boot` 애플리케이션이 `Tomcat` 위에서 실행되듯이, `Debezium`과 같은 '커넥터(Connector)'들은 `Kafka Connect`라는 런타임 위에서 실행됩니다.

Kafka Connect는 자체적으로 클러스터 모드로 동작하여, 여러 Worker 노드에 커넥터 작업을 분산시키고 장애 발생 시 자동으로 복구하는 등의 고가용성 기능을 제공합니다.

**설치 (Docker Compose를 이용한 로컬 환경):**
`docker-compose.yml` 파일에 Kafka Connect 서비스를 추가합니다.

```yaml
# docker-compose.yml
services:
  # ... (zookeeper, kafka 서비스는 이미 존재) ...

  connect:
    image: confluentinc/cp-kafka-connect:7.5.0
    ports:
      - "8083:8083" # Kafka Connect REST API 포트
    depends_on:
      - kafka
    environment:
      # --- 기본 Kafka Connect 설정 ---
      CONNECT_BOOTSTRAP_SERVERS: "kafka:9092"
      CONNECT_REST_ADVERTISED_HOST_NAME: "connect"
      CONNECT_GROUP_ID: "ecommerce-connect-group"
      # --- 커넥터의 상태, 설정, 오프셋을 저장할 내부 토픽 설정 ---
      CONNECT_CONFIG_STORAGE_TOPIC: "connect-configs"
      CONNECT_OFFSET_STORAGE_TOPIC: "connect-offsets"
      CONNECT_STATUS_STORAGE_TOPIC: "connect-status"
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1 # (개발 환경용)
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      # --- Debezium 커넥터 플러그인 설치를 위한 설정 ---
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"
```

-----

### \#\# Debezium: "DB 트랜잭션 로그를 위한 통역사"

**Debezium**은 `Kafka Connect` 위에서 동작하는 \*\*'소스 커넥터(Source Connector)'\*\*입니다. Debezium은 PostgreSQL의 WAL(Write-Ahead Log), MySQL의 Binlog와 같은 데이터베이스의 트랜잭션 로그를 실시간으로 읽어, 모든 `INSERT`, `UPDATE`, `DELETE` 이벤트를 구조화된 JSON 메시지로 변환한 뒤 Kafka 토픽으로 발행하는 역할을 합니다.

#### 1\. Debezium 커넥터 플러그인 설치

Debezium 커넥터를 사용하려면, 위에서 띄운 `Kafka Connect` 컨테이너에 **Debezium 플러그인 `.jar` 파일들을 추가**해야 합니다. 이를 위한 가장 쉬운 방법은, Debezium 플러그인이 미리 설치된 커스텀 도커 이미지를 만드는 것입니다.

```dockerfile
# Dockerfile.connect
FROM confluentinc/cp-kafka-connect:7.5.0

# Debezium PostgreSQL 커넥터 2.5.0.Final 버전 다운로드 및 설치
RUN confluent-hub install --no-prompt debezium/debezium-connector-postgresql:2.5.0.Final
```

이 `Dockerfile`로 이미지를 빌드한 뒤, `docker-compose.yml`의 `connect` 서비스가 이 커스텀 이미지를 사용하도록 변경합니다.

#### 2\. Debezium 커넥터 등록

`Kafka Connect`가 실행되면, 우리는 그 REST API(8083 포트)를 통해 어떤 DB를 감시할지 Debezium 커넥터 설정을 \*\*등록(Register)\*\*할 수 있습니다.

다음과 같은 JSON 파일을 작성합니다.

```json
// register-postgres-connector.json
{
  "name": "product-service-pg-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "database.hostname": "postgres-db",
    "database.port": "5432",
    "database.user": "user",
    "database.password": "password",
    "database.dbname": "orderdb",
    "database.server.name": "ecommerce-db-server",
    "table.include.list": "public.products,public.outbox",
    "plugin.name": "pgoutput",
    "topic.prefix": "cdc",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false"
  }
}
```

  * **`connector.class`**: Debezium PostgreSQL 커넥터를 사용하겠다고 명시합니다.
  * **`database.*`**: 감시할 PostgreSQL 데이터베이스의 접속 정보를 입력합니다. (Docker Compose 네트워크이므로, 서비스 이름 `postgres-db`를 사용)
  * **`table.include.list`**: **매우 중요.** 이 DB 안의 모든 테이블이 아닌, 우리가 감시하려는 테이블 목록(`products`와 `outbox`)을 명시합니다.
  * **`topic.prefix`**: Debezium이 생성할 Kafka 토픽의 접두사를 지정합니다. 예를 들어, `products` 테이블의 변경 사항은 `cdc.ecommerce-db-server.public.products`라는 토픽으로 발행됩니다.

이제 이 JSON 파일을 `Kafka Connect`에 `POST` 요청으로 전송하여 커넥터를 등록합니다.

```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" \
localhost:8083/connectors/ -d @register-postgres-connector.json
```

이 요청이 성공하면, Debezium은 즉시 PostgreSQL DB에 연결하여 트랜잭션 로그를 읽기 시작합니다. 이제 `products` 테이블에 데이터를 `INSERT`하거나 `UPDATE`하면, 관련 이벤트가 실시간으로 Kafka 토픽에 발행되는 것을 확인할 수 있습니다.

**결론적으로,** Kafka Connect와 Debezium은 애플리케이션의 비즈니스 로직과 데이터 동기화 파이프라인을 완벽하게 분리하는 강력한 인프라를 제공합니다. 개발자는 더 이상 데이터 동기화를 위해 복잡한 코드를 작성할 필요 없이, 자신의 DB에만 집중하면 됩니다. 플랫폼 팀은 이 CDC 파이프라인을 중앙에서 관리하여, 조직 전체의 데이터 흐름을 안정적이고 일관되게 만들어 줍니다.