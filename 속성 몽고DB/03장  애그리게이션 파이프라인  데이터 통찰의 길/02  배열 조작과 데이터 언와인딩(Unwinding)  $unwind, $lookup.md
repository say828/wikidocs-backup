## 02\. 배열 조작과 데이터 언와인딩(Unwinding): $unwind, $lookup

MongoDB의 도큐먼트 모델이 가진 가장 큰 장점 중 하나는 배열을 통해 1:N 관계를 자연스럽게 표현할 수 있다는 것입니다. 하지만 집계 파이프라인에서 이 배열 내부의 각 항목을 개별적으로 분석해야 할 때가 많습니다. 예를 들어, 하나의 주문(document)에 포함된 여러 상품(array) 각각의 매출을 계산해야 하는 경우입니다. 이때 필요한 것이 바로 배열을 조작하는 강력한 스테이지, `$unwind`와 `$lookup`입니다.

-----

### `$unwind`: 배열을 개별 도큐먼트로 펼치기

`$unwind` 스테이지는 입력받은 도큐먼트에서 지정된 배열 필드를 해체(deconstruct)하여, 배열의 **각 요소에 대해** 도큐먼트를 하나씩 복제하여 출력합니다. 마치 한 상자에 담긴 여러 개의 상품을 각각 별개의 송장으로 발행하는 것과 같습니다.

  * **동작 방식:** `items` 배열에 3개의 상품이 들어있는 주문 도큐먼트가 `$unwind` 스테이지를 통과하면, `items` 필드의 값만 각기 다른 3개의 주문 도큐먼트로 변환되어 다음 스테이지로 전달됩니다. 나머지 필드(주문 번호, 고객 ID 등)는 그대로 복제됩니다.

**이커머스 예시: `orders` 컬렉션의 `items` 배열을 펼쳐서 각 판매 항목을 개별 라인으로 만들기**

  * **원본 `order` 도큐먼트:**
    ```json
    {
      _id: "ORD001",
      customerId: "user001",
      items: [
        { product_id: "MDB_MUG_01", quantity: 2, price: 15.99 },
        { product_id: "MDB_TSHIRT_01", quantity: 1, price: 25.00 }
      ]
    }
    ```
  * **파이프라인 스테이지:**
    ```javascript
    { $unwind: "$items" }
    ```
  * **`$unwind` 이후의 결과 (2개의 도큐먼트):**
    ```json
    [
      { _id: "ORD001", customerId: "user001", items: { product_id: "MDB_MUG_01", quantity: 2, price: 15.99 } },
      { _id: "ORD001", customerId: "user001", items: { product_id: "MDB_TSHIRT_01", quantity: 1, price: 25.00 } }
    ]
    ```
    이제 우리는 각 상품 항목(`items`)에 대해 직접 접근하고 그룹화할 수 있는 상태가 되었습니다.

-----

### `$lookup`: 다른 컬렉션의 데이터와 결합하기

`$lookup` 스테이지는 MongoDB 네이티브 방식으로 \*\*두 컬렉션 간의 `LEFT OUTER JOIN`\*\*을 수행합니다. 현재 파이프라인을 통과하고 있는 도큐먼트를 기준으로, 다른 컬렉션에서 관련 있는 정보를 찾아와서 새로운 배열 필드로 추가해 줍니다.

  * **구문:**
      * `from`: 조인할 대상 컬렉션의 이름입니다.
      * `localField`: 현재 파이프라인에 있는 도큐먼트의 조인 키 필드입니다.
      * `foreignField`: `from` 컬렉션에 있는 도큐먼트의 조인 키 필드입니다.
      * `as`: 조인된 정보가 담길 새로운 배열 필드의 이름입니다.

**이커머스 예시: 위에서 `$unwind`된 각 판매 항목에 `products` 컬렉션의 상세 정보(예: 카테고리)를 추가하기**

`$unwind`를 거친 각 도큐먼트에는 `items.product_id` 필드가 있습니다. 이 필드를 `products` 컬렉션의 `_id`와 연결하여 상품의 전체 정보를 가져올 수 있습니다.

```javascript
// ... $unwind 스테이지 이후
{
  $lookup: {
    from: "products",                 // products 컬렉션과 조인
    localField: "items.product_id",   // 현재 도큐먼트의 조인 키
    foreignField: "_id",              // products 컬렉션의 조인 키
    as: "productDetails"            // 결과를 productDetails 배열에 저장
  }
}
```

  * **`$lookup` 이후의 결과:**
    ```json
    {
      _id: "ORD001",
      customerId: "user001",
      items: { product_id: "MDB_MUG_01", ... },
      productDetails: [ // _id가 "MDB_MUG_01"인 상품 도큐먼트가 배열에 담겨 추가됨
        { _id: "MDB_MUG_01", name: "MongoDB Developer Mug", category: "kitchen-goods", ... }
      ]
    }
    ```

### 조합하여 사용하기: 카테고리별 총 매출액 계산

이제 이 두 스테이지를 핵심 스테이지들과 조합하여 "**카테고리별 총 매출액**"을 계산하는 완전한 파이프라인을 만들어 보겠습니다.

```javascript
db.orders.aggregate([
  // 1. 분석에 필요한 '완료' 상태의 주문만 필터링
  { $match: { status: "COMPLETED" } },

  // 2. 주문 내 상품 배열을 개별 도큐먼트로 펼치기
  { $unwind: "$items" },

  // 3. 각 상품 항목에 해당하는 상품 상세 정보를 products 컬렉션에서 가져오기
  {
    $lookup: {
      from: "products",
      localField: "items.product_id",
      foreignField: "_id",
      as: "productInfo"
    }
  },
  
  // 4. $lookup 결과는 배열이므로, 사용하기 쉽게 다시 펼치기
  // (일반적으로 1:1 관계이므로 요소가 하나인 배열이 생성됨)
  { $unwind: "$productInfo" },

  // 5. 상품 카테고리별로 그룹화하고, 각 상품의 (수량 * 가격)의 합계를 계산
  {
    $group: {
      _id: "$productInfo.category", // 상품 카테고리로 그룹화
      totalSales: {
        $sum: { $multiply: ["$items.quantity", "$items.price"] }
      }
    }
  },

  // 6. 총 매출액이 높은 순으로 정렬
  { $sort: { totalSales: -1 } }
])
```

이처럼 `$unwind`와 `$lookup`은 임베딩된 데이터 모델의 장점을 유지하면서도, 필요할 때 관계형 데이터베이스처럼 데이터를 연결하고 분석할 수 있게 해주는 매우 강력한 도구입니다. 다음 절에서는 이보다 더 복잡한 데이터 조작을 가능하게 하는 고급 스테이지들을 살펴보겠습니다.