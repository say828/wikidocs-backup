## 03\. 고급 스테이지 활용: $facet, $graphLookup

핵심 스테이지들만으로도 대부분의 집계 요구사항을 해결할 수 있지만, 때로는 더 복잡하고 다각적인 데이터 분석이 필요합니다. MongoDB는 이러한 시나리오를 위해 `$facet`과 `$graphLookup`과 같은 강력한 고급 스테이지를 제공합니다. 이 스테이지들은 단일 쿼리 내에서 여러 분석을 동시에 수행하거나, 계층적인 데이터를 재귀적으로 탐색하는 등 한 차원 높은 데이터 처리 능력을 보여줍니다.

-----

### `$facet`: 단일 쿼리로 여러 관점의 리포트 생성하기

`$facet` 스테이지는 **단일 입력 데이터셋에 대해, 서로 독립적인 여러 개의 애그리게이션 파이프라인을 병렬로 실행**할 수 있게 해줍니다. 각 하위 파이프라인의 결과는 최종 출력 도큐먼트의 별도 필드에 담겨 반환됩니다.

**💡 언제 사용하는가?**
이커머스 사이트의 검색 결과 페이지를 생각해 보세요. 사용자가 "노트북"을 검색하면, 결과 목록과 함께 사이드바에 [카테고리별 상품 수], [브랜드별 상품 수], [가격대별 상품 수]와 같은 정보가 한 번에 표시됩니다. `$facet`을 사용하면, 이 모든 다른 관점의 집계 정보를 단 한 번의 데이터베이스 쿼리로 생성할 수 있습니다.

**이커머스 예시: `products` 컬렉션에 대해 세 가지 다른 분석을 동시에 수행**

1.  **categories:** 카테고리별 상품 수 계산
2.  **priceRanges:** 가격대별 상품 수 계산
3.  **avgPrice:** 전체 상품의 평균 가격 계산

<!-- end list -->

```javascript
db.products.aggregate([
  {
    $facet: {
      // 파이프라인 1: 카테고리별 개수
      "categories": [
        { $group: { _id: "$category", count: { $sum: 1 } } },
        { $sort: { count: -1 } }
      ],
      // 파이프라인 2: 가격대별 개수 ($bucket 스테이지 활용)
      "priceRanges": [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 100, 500, 1000, 5000],
            default: "Over 5000",
            output: { count: { $sum: 1 } }
          }
        }
      ],
      // 파이프라인 3: 전체 평균 가격
      "averagePrice": [
        { $group: { _id: null, avgPrice: { $avg: "$price" } } }
      ]
    }
  }
])
```

  * **`$facet`의 결과 (단일 도큐먼트):**
    ```json
    {
      "categories": [ { "_id": "electronics", "count": 120 }, ... ],
      "priceRanges": [ { "_id": 100, "count": 75 }, ... ],
      "averagePrice": [ { "_id": null, "avgPrice": 895.50 } ]
    }
    ```
    이처럼 `$facet`은 여러 번의 데이터베이스 왕복 없이도 풍부한 분석 결과를 제공하여, 대시보드나 복잡한 리포팅 화면을 구성할 때 매우 효율적입니다.

-----

### `$graphLookup`: 계층 및 그래프 데이터 탐색

`$graphLookup` 스테이지는 컬렉션 내에서 재귀적인 검색을 수행하여 계층 구조나 그래프 관계를 탐색합니다. 특정 도큐먼트에서 시작하여, 지정된 필드를 통해 연결된 다른 도큐먼트들을 연쇄적으로 찾아 나갑니다.

**💡 언제 사용하는가?**

  * 조직도에서 특정 직원의 모든 상위 보고 라인 찾기
  * 이커머스의 상품 카테고리 계층 구조 탐색 (예: "노트북" -\> "컴퓨터" -\> "전자제품")
  * 소셜 네트워크에서 '친구의 친구' 관계망 찾기

**이커머스 예시: `categories` 컬렉션에서 'SSD' 카테고리의 상위 카테고리 전체 경로 찾기**

  * **`categories` 컬렉션 샘플 데이터:**

    ```json
    [
      { "_id": "ROOT", "name": "전체", "parent": null },
      { "_id": "PC_PARTS", "name": "PC 부품", "parent": "ROOT" },
      { "_id": "STORAGE", "name": "저장장치", "parent": "PC_PARTS" },
      { "_id": "SSD", "name": "SSD", "parent": "STORAGE" }
    ]
    ```

  * **파이프라인:**

    ```javascript
    db.categories.aggregate([
      // 1. 'SSD' 카테고리에서 탐색 시작
      { $match: { _id: "SSD" } },
      
      // 2. 'parent' 필드를 따라 재귀적으로 상위 카테고리 탐색
      {
        $graphLookup: {
          from: "categories",           // 동일한 categories 컬렉션 내에서 탐색
          startWith: "$parent",         // 시작 문서의 'parent' 필드 값으로 첫 탐색 시작
          connectFromField: "parent",   // 각 탐색 단계에서 'parent' 필드를 사용
          connectToField: "_id",        // 'parent' 필드 값과 다른 문서의 '_id' 필드를 연결
          as: "categoryPath",           // 결과를 'categoryPath' 배열에 저장
          depthField: "depth"           // 재귀 깊이를 저장할 필드 이름
        }
      }
    ])
    ```

  * **결과:** 'SSD' 도큐먼트에 `categoryPath` 라는 필드가 추가되고, 그 안에는 'STORAGE', 'PC\_PARTS', 'ROOT' 도큐먼트가 재귀 깊이와 함께 담겨있게 됩니다. 이를 통해 "SSD \> 저장장치 \> PC 부품 \> 전체" 라는 전체 계층 경로를 한 번의 쿼리로 알아낼 수 있습니다.

이러한 고급 스테이지들은 애그리게이션 파이프라인의 표현력을 한 단계 끌어올려, 복잡한 비즈니스 인텔리전스 요구사항을 데이터베이스 단에서 직접 해결할 수 있는 강력한 무기를 제공합니다.