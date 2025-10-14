# 02장: MongoDB 쿼리 언어(MQL) 완전 정복

앞선 1장에서 우리는 견고한 애플리케이션의 뼈대가 되는 데이터 모델링의 원칙과 패턴을 깊이 있게 탐구했습니다. 이제 잘 설계된 데이터 구조 위에, 데이터를 자유자재로 다루는 언어인 **MongoDB 쿼리 언어(MQL, MongoDB Query Language)** 를 배울 차례입니다. MQL은 풍부한 계층 구조를 가진 도큐먼트 데이터를 위해 특별히 설계된, 강력하고 표현력 넘치는 언어입니다. 이번 장을 통해 여러분은 단순한 데이터 조회를 넘어, 복잡한 비즈니스 로직을 쿼리 하나로 구현하는 MQL 전문가로 거듭나게 될 것입니다.

-----

## 00\. CRUD 연산의 기본과 철학

모든 데이터베이스 상호작용의 근간은 네 가지 기본 연산, 즉 **CRUD(Create, Read, Update, Delete)** 로 이루어집니다. MQL의 CRUD 연산은 SQL과 같은 문자열 기반의 언어가 아닌, 데이터와 동일한 **도큐먼트(Document) 기반의 구조**를 사용한다는 점에서 근본적인 차이가 있습니다. 이는 개발자가 마치 애플리케이션에서 JSON 객체를 다루듯, 직관적으로 데이터베이스와 소통할 수 있게 해주는 MQL의 핵심 철학입니다.

### Create: 데이터 생성 (`insertOne`, `insertMany`)

새로운 도큐먼트를 컬렉션에 추가하는 연산입니다.

  * **`insertOne(document)`**: 단 하나의 도큐먼트를 삽입합니다.
    `_id` 필드를 명시하지 않으면, MongoDB는 고유한 `ObjectId` 값을 자동으로 생성하여 추가합니다. 이것이 일반적인 사용법입니다.

    ```javascript
    // 이커머스의 products 컬렉션에 새 상품 추가
    db.products.insertOne({
        _id: "MDB_MUG_01",
        name: "MongoDB Developer Mug",
        category: "kitchen-goods",
        price: 15.99,
        stock: 200,
        tags: ["mug", "developer", "office"],
        createdAt: new Date()
    });
    ```

  * **`insertMany([document1, document2, ...])`**: 여러 도큐먼트를 배열에 담아 한 번의 요청으로 삽입합니다. `insertOne`을 반복문으로 호출하는 것보다 네트워크 비용이 절감되어 훨씬 효율적입니다.

    ```javascript
    // 하나의 상품에 대한 여러 리뷰를 reviews 컬렉션에 추가
    db.reviews.insertMany([
        { product_id: "MDB_MUG_01", username: "atlas_dev", rating: 5, comment: "Perfect for my morning coffee!" },
        { product_id: "MDB_MUG_01", username: "bson_lover", rating: 4, comment: "A bit smaller than I expected." }
    ]);
    ```

### Read: 데이터 읽기 (`findOne`, `find`)

컬렉션에서 조건에 맞는 도큐먼트를 조회하는 연산입니다.

  * **`findOne(query, projection)`**: 조건(`query`)에 맞는 **첫 번째 도큐먼트 하나**를 반환합니다. 결과가 없으면 `null`을 반환합니다.

    ```javascript
    // _id가 "MDB_MUG_01"인 상품 하나를 조회
    db.products.findOne({ _id: "MDB_MUG_01" });
    ```

  * **`find(query, projection)`**: 조건(`query`)에 맞는 **모든 도큐먼트를 가리키는 커서(Cursor)** 를 반환합니다. 커서는 결과셋에 대한 포인터로, 실제 데이터는 애플리케이션이 커서를 순회할 때 메모리에 로드됩니다. 이는 수백만 건의 문서를 한 번에 불러와 발생하는 메모리 문제를 방지하는 효율적인 방식입니다. `query`를 빈 문서 `{}`로 전달하면 컬렉션의 모든 도큐먼트를 조회합니다.

    ```javascript
    // category가 "kitchen-goods"인 모든 상품을 조회
    const cursor = db.products.find({ category: "kitchen-goods" });
    cursor.forEach(doc => {
        printjson(doc);
    });
    ```

### Update: 데이터 수정 (`updateOne`, `updateMany`)

기존 도큐먼트의 내용을 변경하는 연산입니다. MQL의 업데이트는 도큐먼트 전체를 교체하는 대신, **업데이트 연산자(`$set`, `$inc` 등)** 를 사용하여 필드 단위의 원자적(Atomic) 변경을 수행하는 것이 핵심입니다.

  * **`updateOne(filter, update)`**: 조건(`filter`)에 맞는 **첫 번째 도큐먼트 하나**를 수정합니다.

    ```javascript
    // "MDB_MUG_01" 상품의 가격을 18.99로, 재고를 150으로 변경
    db.products.updateOne(
        { _id: "MDB_MUG_01" },
        { $set: { price: 18.99, stock: 150 } }
    );
    ```

  * **`updateMany(filter, update)`**: 조건(`filter`)에 맞는 **모든 도큐먼트**를 수정합니다.

    ```javascript
    // 모든 "kitchen-goods" 카테고리 상품에 "sale" 태그를 추가
    db.products.updateMany(
        { category: "kitchen-goods" },
        { $addToSet: { tags: "sale" } } // $addToSet은 배열에 중복 값이 없을 때만 추가
    );
    ```

### Delete: 데이터 삭제 (`deleteOne`, `deleteMany`)

컬렉션에서 도큐먼트를 영구적으로 제거하는 연산입니다.

  * **`deleteOne(filter)`**: 조건(`filter`)에 맞는 **첫 번째 도큐먼트 하나**를 삭제합니다.

    ```javascript
    // 특정 리뷰 하나를 ID로 찾아 삭제
    db.reviews.deleteOne({ _id: ObjectId("67f89b2b8e3a1f9d2d1b4b1a") });
    ```

  * **`deleteMany(filter)`**: 조건(`filter`)에 맞는 **모든 도큐먼트**를 삭제합니다. 매우 강력한 연산이므로 사용에 각별한 주의가 필요합니다. `{}`를 조건으로 전달하면 컬렉션의 모든 데이터가 삭제됩니다.

    ```javascript
    // 재고가 0인 모든 상품을 삭제
    db.products.deleteMany({ stock: 0 });
    ```

이 네 가지 CRUD 연산은 MQL의 가장 기본적인 출발점입니다. 이어지는 절에서는 각 연산, 특히 `find`와 `update`에서 사용할 수 있는 다채롭고 강력한 연산자들을 깊이 있게 탐구하여, MongoDB의 진정한 잠재력을 끌어내는 방법을 배우게 될 것입니다.