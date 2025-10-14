## MongoDB 설정 파일(mongod.conf) 완벽 가이드

`mongod.conf` 파일은 `mongod` 서버 프로세스가 시작될 때 자신의 동작 방식을 결정하는 핵심 설정 파일입니다. YAML 형식을 사용하는 이 파일에 어떤 설정을 하느냐에 따라 데이터베이스의 성능, 보안, 안정성이 크게 달라질 수 있습니다.

이 부록에서는 프로덕션 환경 구성을 위해 반드시 알아야 할 주요 설정 항목들을 섹션별로 나누어 설명하고, 최종적으로 추천 설정 예시를 제공합니다.

-----

### `systemLog`: 로그 설정

> 시스템의 상태를 추적하고 문제를 진단하기 위한 로그 파일의 위치와 방식을 지정합니다.

  * `destination`: 로그 출력 위치. 프로덕션에서는 반드시 `file`로 지정합니다.
  * `path`: 로그 파일의 전체 경로. (예: `/var/log/mongodb/mongod.log`)
  * `logAppend`: `true`로 설정하면 서버 재시작 시 기존 로그 파일에 내용을 추가합니다. `false`일 경우 덮어씁니다. 프로덕션에서는 `true`를 권장합니다.
  * `logRotate`: 로그 파일 로테이션 방식. `mongod` 재시작 없이 로그 파일을 교체할 수 있는 `reopen`을 주로 사용합니다.

-----

### `storage`: 데이터 저장 설정

> 데이터 파일의 위치, 스토리지 엔진, 저널 등 물리적 데이터 저장 방식을 제어합니다.

  * `dbPath`: MongoDB의 모든 데이터 파일이 저장될 디렉토리 경로. **가장 중요한 설정**입니다. (예: `/var/lib/mongo`)
  * `journal.enabled`: `true`로 설정하여 재해 복구를 위한 저널링을 활성화합니다. 프로덕션 환경에서는 **반드시 `true`여야 합니다.** (기본값)
  * `engine`: 사용할 스토리지 엔진. 현대 MongoDB에서는 `wiredTiger`가 기본값이며 유일한 선택지입니다.
  * `wiredTiger`: WiredTiger 엔진에 대한 세부 튜닝 옵션입니다.
      * `engineConfig.cacheSizeGB`: WiredTiger 내부 캐시의 최대 크기를 GB 단위로 지정합니다. 서버 RAM 용량에 맞춰 적절히 설정하는 것이 성능에 매우 중요합니다.
      * `collectionConfig.blockCompressor`: 컬렉션 데이터에 사용할 기본 압축 알고리즘. `snappy`(기본값), `zlib`, `zstd` 중에서 워크로드에 맞게 선택합니다.
      * `indexConfig.prefixCompression`: 인덱스 데이터에 접두사 압축을 사용할지 여부. `true`(기본값)로 설정하면 인덱스 크기를 줄이는 데 도움이 됩니다.

-----

### `net`: 네트워크 설정

> 클라이언트 및 다른 서버와의 통신 방식을 설정합니다.

  * `port`: `mongod`가 리슨할 TCP 포트. 기본값은 `27017`입니다.
  * `bindIp`: 서버가 연결을 허용할 IP 주소 목록. **보안상 매우 중요한 설정**입니다.
      * `127.0.0.1`: 로컬호스트에서의 연결만 허용합니다.
      * 프로덕션 환경에서는 `127.0.0.1`과 함께 애플리케이션 서버, 다른 레플리카 셋 멤버 등 필요한 서버의 **내부 IP 주소**만 쉼표로 구분하여 명시해야 합니다.
      * **경고:** `0.0.0.0`은 모든 네트워크 인터페이스로부터의 연결을 허용하므로, 방화벽 설정이 완벽하지 않다면 데이터베이스가 외부에 노출될 수 있어 절대로 사용해서는 안 됩니다.
  * `tls`: 네트워크 암호화(TLS/SSL) 설정입니다.
      * `mode`: `requireTLS`로 설정하여 모든 클라이언트가 암호화된 연결을 사용하도록 강제해야 합니다.
      * `certificateKeyFile`: 서버의 TLS 인증서와 개인 키가 포함된 `.pem` 파일의 경로.

-----

### `security`: 보안 설정

> 인증, 권한 부여 등 보안 관련 기능을 활성화합니다.

  * `authorization`: `enabled`로 설정하여 사용자 인증(Authentication) 및 역할 기반 권한 부여(RBAC)를 활성화합니다. **프로덕션 환경에서는 필수입니다.**
  * `clusterAuthMode`: 클러스터 내부 멤버 간의 인증 방식. `keyFile`로 설정합니다.
  * `keyFile`: 클러스터 멤버 간 인증에 사용할 키 파일의 경로. 모든 멤버는 동일한 키 파일을 공유해야 합니다.

-----

### `replication`: 복제 설정

> 레플리카 셋 구성을 위한 설정입니다.

  * `replSetName`: 이 서버가 속할 레플리카 셋의 이름을 지정합니다. 이 설정이 있어야 서버가 레플리카 셋의 멤버로 동작합니다.
  * `oplogSizeMB`: Oplog의 최대 크기를 메가바이트(MB) 단위로 지정합니다. 쓰기 작업이 많은 프로덕션 환경에서는 기본값보다 충분히 크게 설정하여 복제 지연 시 Oplog Window가 짧아지는 문제를 방지해야 합니다.

-----

### 종합 예제: 프로덕션 레플리카 셋 멤버 `mongod.conf`

```yaml
# mongod.conf (프로덕션 레플리카 셋 멤버 추천 설정 예시)

# 데이터 저장 경로
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8 # 예: 16GB RAM 서버에서 캐시를 8GB로 설정

# 로그 파일 설정
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# 네트워크 인터페이스
net:
  port: 27017
  bindIp: 127.0.0.1,192.168.1.101 # 로컬호스트와 내부망 IP만 허용
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem

# 보안 설정
security:
  authorization: enabled
  clusterAuthMode: keyFile
  keyFile: /opt/mongo/mongodb-keyfile

# 복제 설정
replication:
  replSetName: "rs0"
  oplogSizeMB: 4096 # 예: Oplog 크기를 4GB로 설정
```