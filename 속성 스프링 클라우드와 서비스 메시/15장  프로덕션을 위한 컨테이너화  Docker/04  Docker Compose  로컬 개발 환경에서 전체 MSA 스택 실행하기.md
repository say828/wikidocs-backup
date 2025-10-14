## Docker Compose: 로컬 개발 환경에서 전체 MSA 스택 실행하기

15장 03절까지 우리는 Buildpacks를 이용해 `member-service` 같은 **개별 마이크로서비스**를 프로덕션급 컨테이너 이미지로 만드는 방법을 배웠습니다. 하지만 우리 시스템은 수많은 컴포넌트의 집합체입니다.

  * `discovery-service` (Eureka)
  * `config-service`
  * `gateway-service`
  * `member-service`, `product-service`, `order-service`
  * `PostgreSQL` (주문 DB)
  * `Kafka` & `Zookeeper`
  * `Redis`

개발자 한 명이 이 모든 컴포넌트를 매번 올바른 순서대로, 올바른 환경 변수를 설정하여 실행하는 것은 거의 불가능에 가깝습니다. 이 '로컬 개발 환경의 오케스트라'를 지휘하기 위한 도구가 바로 **Docker Compose**입니다.

-----

### Docker Compose: 멀티 컨테이너 애플리케이션의 지휘자 🎼

**Docker Compose**는 **여러 개의 컨테이너**로 구성된 애플리케이션 스택을 **하나의 `YAML` 파일**로 정의하고, **단일 명령어**로 실행 및 관리할 수 있게 해주는 도구입니다.

  * **`docker-compose.yml` 파일:** 전체 시스템을 구성하는 모든 서비스, 데이터베이스, 메시지 큐 등의 '악보' 역할을 합니다. 각 컴포넌트가 어떤 이미지를 사용하고, 어떤 포트를 노출하며, 어떤 환경 변수를 필요로 하는지, 그리고 서비스 간의 의존 관계는 어떠한지를 모두 이 파일 하나에 정의합니다.
  * **`docker-compose up`:** 지휘자가 지휘봉을 휘두르는 것처럼, 이 명령어 하나로 `docker-compose.yml`에 정의된 모든 컨테이너가 의존 순서에 따라 실행되고, 내부 네트워크로 연결됩니다.
  * **`docker-compose down`:** 모든 컨테이너를 한 번에 중지하고 관련 네트워크까지 깔끔하게 제거합니다.

-----

### 실전 예제: `docker-compose.yml`로 전체 MSA 스택 정의하기

이제 우리 이커머스 프로젝트 전체를 로컬에서 실행하기 위한 `docker-compose.yml` 파일을 프로젝트의 루트 디렉토리에 작성해 봅시다.

```yaml
# docker-compose.yml (프로젝트 루트)

# 1. Docker Compose 파일 버전 정의
version: '3.8'

# 2. 실행할 컨테이너(서비스)들의 묶음 정의
services:
  # --- 인프라스트럭처 서비스 ---
  discovery-service:
    # 3. Buildpacks로 미리 빌드해 둔 이미지를 사용
    image: ecommerce/discovery-service:0.0.1
    ports:
      - "8761:8761"

  config-service:
    image: ecommerce/config-service:0.0.1
    ports:
      - "8888:8888"
    depends_on: # discovery-service가 먼저 실행된 후에 실행
      - discovery-service
    environment:
      # 4. 서비스 이름으로 다른 컨테이너에 접근 가능!
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://discovery-service:8761/eureka

  postgres-db:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=orderdb
    volumes: # DB 데이터를 영속적으로 보관하기 위한 볼륨
      - postgres-data:/var/lib/postgresql/data

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    ports:
      - "2181:2181"
    environment:
      - ZOOKEEPER_CLIENT_PORT=2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    environment:
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
      - KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1
  
  # --- 애플리케이션 서비스 ---
  gateway-service:
    image: ecommerce/gateway-service:0.0.1
    ports:
      - "8080:8080"
    depends_on:
      - discovery-service
      - config-service
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://discovery-service:8761/eureka
      - SPRING_CLOUD_CONFIG_URI=http://config-service:8888
      - SPRING_PROFILES_ACTIVE=local

  member-service:
    image: ecommerce/member-service:0.0.1
    ports:
      - "8081:8081" # 로컬에서 디버깅을 위해 포트를 노출할 수 있음
    depends_on:
      - discovery-service
      - config-service
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://discovery-service:8761/eureka
      - SPRING_CLOUD_CONFIG_URI=http://config-service:8888
      - SPRING_PROFILES_ACTIVE=local

# 3. 데이터베이스 데이터를 보존하기 위한 명명된 볼륨 정의
volumes:
  postgres-data:
```

  * **서비스 이름으로 통신:** Docker Compose는 정의된 모든 서비스를 같은 **가상 네트워크**로 묶어줍니다. 따라서 `gateway-service`는 `http://localhost:8761`이 아닌, **서비스 이름**인 `http://discovery-service:8761`로 Eureka 서버에 접근할 수 있습니다. 이는 컨테이너의 IP가 매번 바뀌더라도 통신이 가능하게 해주는 매우 강력한 기능입니다.

-----

### 로컬 개발 워크플로우

이제 개발자의 로컬 개발 환경 구성이 놀랍도록 단순해집니다.

1.  **[1회성]** 각 마이크로서비스 프로젝트 디렉토리로 이동하여 Buildpacks로 이미지를 빌드합니다.
    ```bash
    cd member-service && ./gradlew bootBuildImage --imageName=ecommerce/member-service:0.0.1
    cd ../gateway-service && ./gradlew bootBuildImage --imageName=ecommerce/gateway-service:0.0.1
    # ... 모든 서비스에 대해 반복
    ```
2.  프로젝트 루트 디렉토리에서 다음 명령어 하나만 실행합니다.
    ```bash
    # -d 옵션: 백그라운드에서 실행
    docker-compose up -d
    ```
3.  잠시 후, `docker-compose ps` 명령어로 모든 컨테이너가 정상적으로 실행 중인지 확인합니다.
4.  이제 개발자는 `http://localhost:8080` (게이트웨이)을 통해 전체 MSA 시스템을 테스트하고 개발할 수 있습니다.
5.  개발이 끝나면 다음 명령어로 모든 컨테이너와 네트워크를 깔끔하게 종료합니다.
    ```bash
    docker-compose down
    ```

Docker Compose는 복잡한 MSA의 로컬 개발 환경 문제를 해결하는 필수 도구입니다. 모든 팀원이 동일하고 재현 가능한 환경에서 개발할 수 있도록 하여 생산성을 극대화합니다.

이것으로 15장을 마칩니다. 이제 우리의 모든 서비스는 컨테이너화되었고, 로컬에서 오케스트레이션할 준비까지 마쳤습니다. 다음 16장에서는 이 컨테이너들을 로컬 개발 환경이 아닌, 실제 프로덕션 환경에서 관리하기 위한 \*\*컨테이너 오케스트레이션의 표준, 쿠버네티스(Kubernetes)\*\*의 세계로 떠나보겠습니다.