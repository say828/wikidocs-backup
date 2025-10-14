# 부록

---

## A. 코틀린 표준 라이브러리 핵심 함수 레퍼런스

코틀린의 힘은 간결한 문법뿐만 아니라, 개발자가 자주 사용하는 거의 모든 기능을 미리 구현해 둔 방대하고 강력한 **표준 라이브러리(Standard Library)**에서 나옵니다. 관용적인 코틀린 코드는 이 표준 라이브러리 함수들을 물 흐르듯 자연스럽게 사용하는 코드입니다.

이 부록은 표준 라이브러리의 모든 함수를 나열하는 대신, 코틀린 개발자라면 매일 사용하게 될 가장 핵심적이고 유용한 함수들을 선별하여 요약한 '치트 시트(cheat sheet)'입니다.

---

### 1. 스코프 함수 (Scope Functions)

객체의 컨텍스트 내에서 코드 블록을 실행하여 유창한(fluent) API를 만듭니다. (19장 참고)

| 함수 | 컨텍스트 객체 | 반환 값 | 핵심 용도 |
| :--- | :---: | :---: | :--- |
| **`let`** | `it` | 람다 결과 | **널-안전 호출** 및 값 변환 (`?.let`) |
| **`run`** | `this` | 람다 결과 | 객체에 대한 연산 후 결과 반환 (널-안전 호출 가능) |
| **`with`** | `this` | 람다 결과 | `run`의 비-확장 함수 버전 (널이 아닌 객체 대상) |
| **`apply`** | `this` | **컨텍스트 객체** | **객체 생성 및 초기 설정** (빌더 패턴) |
| **`also`** | `it` | **컨텍스트 객체** | **부가 작업** (로깅, 디버깅 등 사이드 이펙트) |

---

### 2. 컬렉션 핵심 연산 (Core Collection Operations)

#### 변환 (Transformation)

* **`map { (T) -> R }: List<R>`**
    * **용도:** 컬렉션의 모든 요소를 람다를 통해 1:1로 변환하여, 그 결과값들로 구성된 **새로운 리스트**를 반환합니다.
    * **예시:** `val lengths = words.map { it.length }` (문자열 리스트 -> 길이(Int) 리스트)

* **`flatMap { (T) -> Iterable<R> }: List<R>`**
    * **용도:** 모든 요소를 람다를 통해 1:N (컬렉션)으로 변환한 뒤, 이 모든 하위 컬렉션들을 **하나의 평탄화된 리스트**로 합쳐 반환합니다.
    * **예시:** `val allBooks = authors.flatMap { it.books }` (작가 리스트 -> 모든 책 리스트)

#### 필터링 및 조회 (Filtering & Finding)

* **`filter { (T) -> Boolean }: List<T>`**
    * **용도:** 람다의 조건식이 `true`를 반환하는 요소**들만** 모아서 **새로운 리스트**를 반환합니다.
    * **예시:** `val adults = users.filter { it.age > 19 }`

* **`first() / firstOrNull() { (T) -> Boolean }`**
    * **용도:** 람다 조건을 만족하는 **첫 번째 요소**를 반환합니다.
    * `first()`: 조건에 맞는 요소가 없으면 `NoSuchElementException` 예외 발생.
    * `firstOrNull()`: 조건에 맞는 요소가 없으면 `null` 반환 (더 안전함).
    * **예시:** `val admin = users.firstOrNull { it.isAdmin }`

* **`find { (T) -> Boolean }`**
    * **용도:** `firstOrNull()`과 기능적으로 동일합니다. 조건에 맞는 첫 번째 요소를 반환하고, 없으면 `null`을 반환합니다.
    * **예시:** `val product = products.find { it.id == "A123" }`

#### 검사 (Predicates)

* **`any { (T) -> Boolean }: Boolean`**
    * **용도:** 컬렉션의 요소 중 **하나라도** 조건을 만족하면 `true`를 반환합니다.
    * **예시:** `val hasErrors = responses.any { it.statusCode != 200 }`

* **`all { (T) -> Boolean }: Boolean`**
    * **용도:** 컬렉션의 **모든** 요소가 조건을 만족해야만 `true`를 반환합니다.
    * **예시:** `val allInputsValid = inputs.all { it.isValid() }`

* **`none { (T) -> Boolean }: Boolean`**
    * **용도:** 컬렉션의 **모든** 요소가 조건을 만족하지 **않을** 때 `true`를 반환합니다. (`any`의 정반대)
    * **예시:** `val noGuests = users.none { it.isGuest }`

#### 집계 및 그룹화 (Aggregation & Grouping)

* **`count { (T) -> Boolean }: Int`**
    * **용도:** 람다의 조건을 만족하는 요소의 **개수**를 반환합니다.
    * **예시:** `val pendingOrders = orders.count { it.status == "PENDING" }`

* **`sumOf { (T) -> Int/Long/Double }`**
    * **용도:** 각 요소를 람다를 통해 숫자(Int, Long, Double 등)로 변환한 뒤, 이 숫자들의 **총합**을 반환합니다.
    * **예시:** `val totalRevenue = orders.sumOf { it.price }`

* **`reduce { (R, T) -> R }: R`**
    * **용도:** 컬렉션의 첫 번째 요소를 초기 누적값(acc)으로 사용하여, 람다를 순차적으로 실행하며 값을 하나로 **줄여나갑니다.** (컬렉션이 비어있으면 예외 발생)
    * **예시:** `val product = numbers.reduce { acc, i -> acc * i }` (모든 숫자의 곱)

* **`fold(initial: R) { (R, T) -> R }: R`**
    * **용도:** `reduce`와 동일하지만, 개발자가 명시적인 **초기값(initial)**을 제공합니다. (컬렉션이 비어있어도 초기값을 반환하므로 더 안전함)
    * **예시:** `val csvHeader = columns.fold("Index") { acc, column -> "$acc,$column" }`

* **`groupBy { (T) -> K }: Map<K, List<T>>`**
    * **용도:** 람다가 반환하는 값(Key)을 기준으로 컬렉션을 그룹화하여 **Map**을 생성합니다.
    * **예시:** `val usersByCity = users.groupBy { it.city }` (도시 이름을 Key로, User 리스트를 Value로 갖는 맵 생성)

---

### 3. 유틸리티 함수 (Utility Functions)

* **`repeat(times: Int, action: (Int) -> Unit)`**
    * **용도:** `action` 람다를 `times` 횟수만큼 반복 실행합니다. (0부터 `times-1`까지의 인덱스가 람다로 전달됩니다.)
    * **예시:** `repeat(3) { println("Hello $it") }` (Hello 0, Hello 1, Hello 2 출력)

* **`TODO(reason: String)`**
    * **용도:** "아직 구현되지 않았음"을 명시적으로 나타내는 함수입니다. 이 함수는 호출되면 항상 **`NotImplementedError`** 예외를 던집니다. IDE가 이 함수를 강조 표시하여 구현이 필요한 부분을 쉽게 찾도록 도와줍니다.
    * **예시:** `fun calculateTax() = TODO("세율 정책 확정 후 구현 예정")`