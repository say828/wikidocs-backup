## 04\. 전문가를 위한 실습 환경 구축: Docker, Atlas CLI, Compass

이론을 넘어 실제 코드를 실행하고 데이터를 직접 만져보는 경험은 전문가로 성장하기 위한 필수 과정입니다. 이 절에서는 앞으로 책의 모든 예제를 원활하게 실습할 수 있도록, 일관성 있고 재현 가능한 전문가급 개발 환경을 구축하는 방법을 안내합니다. 우리는 세 가지 핵심 도구를 사용할 것입니다.

  * **Docker:** 로컬 머신에 격리된 MongoDB 서버를 손쉽게 구축하고 관리합니다.
  * **MongoDB Compass:** GUI 환경에서 데이터를 시각적으로 탐색하고 쿼리를 실행합니다.
  * **MongoDB Atlas CLI:** 클라우드 데이터베이스를 터미널에서 효율적으로 제어합니다.

### Docker를 이용한 로컬 MongoDB 구축

내 컴퓨터에 직접 MongoDB를 설치하는 대신 Docker를 사용하면, 운영체제에 상관없이 깨끗하고 격리된 환경을 유지할 수 있으며, 필요에 따라 다양한 버전의 데이터베이스를 손쉽게 생성하고 삭제할 수 있습니다.

먼저 로컬 머신에 [Docker Desktop](https://www.docker.com/products/docker-desktop/)이 설치되어 있어야 합니다.

**1. Standalone 인스턴스 실행**

가장 간단한 단일 MongoDB 인스턴스를 실행하는 명령어입니다. 터미널을 열고 다음을 입력하세요.

```bash
docker run -d --name mongodb-dev \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  mongo
```

  * `-d`: 컨테이너를 백그라운드에서 실행합니다.
  * `--name mongodb-dev`: 컨테이너에 `mongodb-dev`라는 이름을 부여합니다.
  * `-p 27017:27017`: 내 컴퓨터(호스트)의 27017번 포트를 컨테이너의 27017번 포트로 연결합니다.
  * `-e`: 환경 변수를 설정하여 초기 root 계정을 생성합니다. (주의: 로컬 개발용 암호이며, 프로덕션에서는 절대 사용하면 안 됩니다.)
  * `mongo`: 사용할 Docker 이미지 이름입니다.

**2. 컨테이너에 접속하여 `mongosh` 실행**

아래 명령어로 실행 중인 컨테이너 내부의 MongoDB 셸(`mongosh`)에 접속할 수 있습니다.

```bash
docker exec -it mongodb-dev mongosh -u admin -p password
```

### MongoDB Atlas CLI 설치 및 연동

MongoDB Atlas는 MongoDB사가 직접 제공하는 완전 관리형 클라우드 데이터베이스 서비스(DBaaS)입니다. Atlas CLI를 사용하면 웹 UI에 로그인하지 않고도 터미널에서 클러스터를 생성, 관리, 모니터링할 수 있어 자동화에 매우 유용합니다.

1.  **설치:** 사용 중인 운영체제에 맞게 설치합니다. (macOS의 경우)
    ```bash
    brew install mongodb-atlas-cli
    ```
2.  **인증:** 아래 명령어를 실행하면 브라우저가 열리며, Atlas 계정으로 로그인하여 CLI를 인증할 수 있습니다.
    ```bash
    atlas auth login
    ```
3.  **무료 클러스터 생성:** 인증이 완료되면, 다음 명령어로 서울 리전(`AP_NORTHEAST_2`)에 무료 등급(M0) 클러스터를 1분 안에 생성할 수 있습니다.
    ```bash
    atlas clusters create myDevCluster --provider AWS --region AP_NORTHEAST_2 --tier M0
    ```

### MongoDB Compass - GUI 클라이언트

Compass는 MongoDB가 공식적으로 제공하는 GUI 도구로, 데이터베이스 작업을 위한 매우 강력하고 직관적인 인터페이스를 제공합니다.

  * **주요 기능:**
      * 데이터와 스키마의 시각적 탐색
      * GUI 빌더를 통한 쿼리 및 애그리게이션 파이프라인 작성
      * 쿼리 성능 분석을 위한 시각적 `explain` 플랜
      * 인덱스 생성 및 관리
      * 실시간 서버 성능 지표 확인

[MongoDB Compass 다운로드 페이지](https://www.mongodb.com/try/download/compass)에서 자신의 운영체제에 맞는 버전을 설치합니다. 설치 후 실행하여 새 연결(New Connection)을 클릭하고, 앞서 Docker로 생성한 로컬 데이터베이스의 연결 문자열(`mongodb://admin:password@localhost:27017/`)을 입력하여 접속할 수 있습니다. Atlas 클러스터의 경우, Atlas UI에서 제공하는 연결 문자열을 복사하여 붙여넣으면 됩니다.

이제 우리는 로컬 개발과 테스트를 위한 Docker 환경, 클라우드 환경을 제어하는 Atlas CLI, 그리고 생산성을 극대화할 Compass까지 모두 갖추었습니다. 이 강력한 도구들을 기반으로 다음 장부터 MongoDB의 세계를 본격적으로 탐험해 보겠습니다.