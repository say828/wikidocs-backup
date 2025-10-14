## 주요 mongosh 명령어 레퍼런스

이 부록은 `mongosh` 환경에서 데이터베이스 관리 및 데이터 조작을 위해 자주 사용되는 주요 셸 헬퍼 명령어들을 모아놓은 빠른 참조 가이드입니다. 이 목록은 전체가 아니며, 일상적인 개발 및 운영 업무에서 가장 유용한 명령어들을 선별하여 정리했습니다.

-----

### 데이터베이스 및 컬렉션 작업

  * **모든 데이터베이스 목록 보기**

    ```shell
    show dbs
    ```

  * **데이터베이스 컨텍스트 전환**

    > 존재하지 않는 데이터베이스 이름을 사용하면, 첫 데이터 삽입 시 해당 데이터베이스가 자동으로 생성됩니다.

    ```shell
    use ecommerce
    ```

  * **현재 데이터베이스 확인**

    ```shell
    db
    ```

  * **현재 데이터베이스의 모든 컬렉션 목록 보기**

    ```shell
    show collections
    ```

  * **컬렉션 명시적으로 생성**

    > capped 컬렉션이나 document validation 같은 옵션을 적용할 때 사용합니다.

    ```shell
    db.createCollection("logs", { capped: true, size: 10485760 })
    ```

  * **컬렉션 삭제**

    > 컬렉션의 모든 데이터와 인덱스가 영구적으로 삭제됩니다.

    ```shell
    db.products.drop()
    ```

  * **현재 데이터베이스 삭제**

    > 현재 `use`로 선택된 데이터베이스 전체가 영구적으로 삭제됩니다.

    ```shell
    db.dropDatabase()
    ```

-----

### CRUD 및 쿼리 관련

> `insertOne()`, `find()`, `updateOne()`, `deleteMany()` 등 기본적인 CRUD 메소드에 대한 상세 내용은 본문 2장을 참조하십시오.

  * **쿼리 결과 가독성 높여 보기**

    > 최신 `mongosh`는 기본적으로 결과값을 보기 좋게 출력하지만, `.pretty()`는 전통적으로 널리 사용되어 온 메소드입니다.

    ```shell
    db.users.find().pretty()
    ```

  * **조건에 맞는 문서 개수 세기 (권장)**

    ```shell
    db.products.countDocuments({ category: "electronics" })
    ```

  * **컬렉션의 전체 문서 개수 빠르게 추정**

    > 메타데이터를 사용하므로 매우 빠릅니다.

    ```shell
    db.products.estimatedDocumentCount()
    ```

  * **커서 결과 다음 배치 보기**

    > `find()` 실행 후 커서가 반환되었을 때, `it`을 입력하면 다음 결과 배치를 출력합니다.

    ```shell
    it
    ```

-----

### 인덱스 관련

  * **컬렉션의 모든 인덱스 정보 보기**

    ```shell
    db.products.getIndexes()
    ```

  * **새로운 인덱스 생성**

    ```shell
    db.orders.createIndex({ customerId: 1, orderDate: -1 })
    ```

  * **특정 인덱스 삭제**

    > 인덱스 이름을 `getIndexes()`로 확인한 후 사용합니다.

    ```shell
    db.orders.dropIndex("customerId_1_orderDate_-1")
    ```

  * **컬렉션의 전체 인덱스 크기 확인 (바이트 단위)**

    ```shell
    db.products.totalIndexSize()
    ```

-----

### 관리 및 진단

  * **서버 버전 확인**
    ```shell
    db.version()
    ```
  * **현재 데이터베이스 통계 보기**
    ```shell
    db.stats()
    ```
  * **특정 컬렉션 통계 보기**
    ```shell
    db.products.stats()
    ```
  * **레플리카 셋 상태 확인**
    > 멤버들의 상태, `optime`, 복제 지연 등을 상세히 보여줍니다.
    ```shell
    rs.status()
    ```
  * **샤드 클러스터 상태 확인**
    > 샤드 목록, 밸런서 상태, 청크 정보 등을 보여줍니다.
    ```shell
    sh.status()
    ```
  * **현재 실행 중인 작업 보기**
    > 느린 쿼리나 오래 실행되는 작업을 진단할 때 필수적입니다.
    ```shell
    db.currentOp()
    ```
  * **특정 작업 강제 종료**
    > `currentOp()`에서 확인한 `opid`를 사용합니다.
    ```shell
    db.killOp(12345)
    ```
  * **데이터베이스 프로파일러 설정**
    > 느린 쿼리를 자동으로 로깅하는 기능을 활성화합니다. (0: off, 1: slow ops, 2: all ops)
    ```shell
    // 100ms보다 오래 걸리는 쿼리를 로깅
    db.setProfilingLevel(1, { slowms: 100 })
    ```