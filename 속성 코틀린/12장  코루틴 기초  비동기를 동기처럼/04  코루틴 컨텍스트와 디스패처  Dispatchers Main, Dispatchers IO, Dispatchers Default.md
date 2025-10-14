### 코루틴 컨텍스트와 디스패처: Dispatchers.Main, Dispatchers.IO, Dispatchers.Default

`launch`나 `async`로 코루틴을 시작할 때, "그래서 이 코루틴은 대체 **어떤 스레드에서 실행되는 걸까?**" 라는 근본적인 질문이 생깁니다. 이 질문에 대한 답을 쥐고 있는 것이 바로 \*\*코루틴 컨텍스트(CoroutineContext)\*\*와 그 안에 포함된 \*\*디스패처(Dispatcher)\*\*입니다.

**코루틴 컨텍스트**는 코루틴이 실행되는 **환경**에 대한 정보의 집합입니다. 코루틴의 이름, `Job`, 예외 핸들러 등 다양한 요소로 구성될 수 있으며, 그중 가장 중요한 요소가 바로 \*\*코루틴 디스패처(CoroutineDispatcher)\*\*입니다.

**디스패처**는 코루틴을 특정 스레드 또는 스레드 풀(Thread Pool)에 \*\*배정(dispatch)\*\*하는 역할을 하는 스케줄러입니다. 즉, 코루틴이 어느 스레드 위에서 실행될지를 결정합니다. ✈️

-----

#### 표준 디스패처: 올바른 도구 선택하기

`kotlinx.coroutines` 라이브러리는 대부분의 상황에 맞춰 미리 최적화된 세 가지 표준 디스패처를 제공합니다. 작업의 성격에 맞는 올바른 디스패처를 선택하는 것은 성능과 응답성에 매우 중요합니다.

디스패처는 `launch`나 `async`와 같은 코루틴 빌더를 호출할 때 파라미터로 전달하여 지정할 수 있습니다.

##### 1\. `Dispatchers.Main`: UI 스레드 전용

  * **목적:** UI를 직접 조작하는 작업을 실행하기 위한 디스패처입니다. 안드로이드, JavaFX, Swing과 같은 UI 프레임워크의 **메인 스레드** 또는 **UI 스레드**에서 코루틴을 실행합니다.
  * **사용 사례:** 텍스트 뷰의 내용을 바꾸거나, 사용자에게 알림을 보여주는 등 화면과 관련된 모든 작업은 반드시 `Dispatchers.Main`에서 수행해야 합니다.
  * **참고:** 이 디스패처는 `kotlinx-coroutines-android`나 `kotlinx-coroutines-javafx`와 같은 플랫폼 의존적인 라이브러리를 추가해야 사용할 수 있습니다.

<!-- end list -->

```kotlin
// 안드로이드 예시
launch(Dispatchers.Main) {
    myTextView.text = "UI가 업데이트되었습니다!"
}
```

##### 2\. `Dispatchers.IO`: 입출력(I/O) 작업 전용

  * **목적:** **네트워킹**, **파일 읽기/쓰기**, **데이터베이스 쿼리**와 같이 입출력(I/O)으로 인해 스레드가 대기(blocking)할 가능성이 높은 작업에 최적화되어 있습니다.
  * **동작 방식:** 필요에 따라 스레드를 생성하고 공유하는 스레드 풀을 사용합니다. 수많은 I/O 작업을 동시에 효율적으로 처리하도록 설계되었습니다.
  * **비유:** 대기 시간이 긴 화물 처리를 위한 거대한 물류 센터 🚚와 같습니다.

<!-- end list -->

```kotlin
launch(Dispatchers.IO) {
    val user = apiClient.fetchUserData() // 네트워크 API 호출
    database.saveUser(user)          // 데이터베이스에 저장
}
```

##### 3\. `Dispatchers.Default`: CPU 집약적 작업 전용

  * **목적:** 리스트 정렬, 복잡한 연산, JSON 파싱, 이미지 처리 등 **CPU를 많이 사용하는** 계산 위주의 작업에 최적화되어 있습니다.
  * **동작 방식:** 컴퓨터의 CPU 코어 수에 맞춰진 고정된 크기의 스레드 풀을 사용합니다. CPU를 한계까지 사용하는 작업은 코어 수 이상의 스레드를 사용해도 성능 향상에 도움이 되지 않기 때문입니다.
  * **비유:** 무거운 연산을 위한 고성능 컴퓨팅 클러스터 💻와 같습니다.

<!-- end list -->

```kotlin
val deferredResult = async(Dispatchers.Default) {
    // 매우 큰 리스트를 정렬하는 CPU 집약적 작업
    veryLargeList.sorted()
}
```

-----

#### 컨텍스트 전환: `withContext`

코루틴은 한 번 시작된 디스패처에 묶여있지 않습니다. 실행 중에 필요에 따라 자유롭게 스레드를 오갈 수 있습니다. 이때 사용하는 가장 표준적인 방법이 바로 **`withContext`** 함수입니다.

`withContext`는 코루틴의 실행 컨텍스트(디스패처)를 일시적으로 다른 것으로 전환하고, 내부의 코드 블록이 완료되면 원래의 컨텍스트로 자동으로 복귀시켜주는 일시 중단 함수입니다.

**가장 일반적인 사용 패턴:**
UI 스레드(`Main`)에서 시작 → 백그라운드 스레드(`IO` 또는 `Default`)로 전환하여 무거운 작업 수행 → 다시 UI 스레드(`Main`)로 돌아와 결과 표시

```kotlin
// 안드로이드 ViewModel에서 실행한다고 가정
fun fetchAndShowUser() {
    viewModelScope.launch { // 기본적으로 Dispatchers.Main에서 시작
        // 1. Main 스레드: 로딩 UI 표시
        showLoadingIndicator()

        // 2. IO 스레드로 전환하여 네트워크 작업 수행
        val user = withContext(Dispatchers.IO) {
            apiClient.fetchUserData()
        }

        // 3. withContext 블록이 끝나면 자동으로 Main 스레드로 복귀!
        //    결과를 UI에 표시
        hideLoadingIndicator()
        myTextView.text = user.name
    }
}
```

`withContext`를 사용하면 콜백 지옥 없이, 동기적인 코드처럼 보이는 깔끔한 코드로 안전하게 백그라운드 작업을 처리하고 UI를 업데이트할 수 있습니다. 이는 코루틴을 사용하는 가장 큰 이유 중 하나입니다.