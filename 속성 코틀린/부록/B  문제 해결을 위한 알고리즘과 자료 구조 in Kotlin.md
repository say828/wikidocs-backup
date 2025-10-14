## B. 문제 해결을 위한 알고리즘과 자료 구조 in Kotlin

이 책의 본문은 코틀린을 사용하여 견고하고 확장성 있는 '애플리케이션'을 구축하는 데 중점을 두었습니다. 하지만 프로그래밍의 또 다른 중요한 축은 '문제 해결(Problem Solving)', 즉 알고리즘입니다. 코틀린은 간결한 문법, 강력한 표준 라이브러리, 그리고 타입 안전성 덕분에 알고리즘 문제 풀이(코딩 테스트, 경쟁 프로그래밍 등)에서도 최고의 언어 중 하나로 각광받고 있습니다.

이 부록에서는 몇 가지 핵심적인 자료 구조와 알고리즘이 코틀린의 관용구를 만나 얼마나 더 깔끔하고 안전하게 구현될 수 있는지 살펴봅니다.

-----

### 1\. 코틀린의 자료 구조 구현

#### 스택(Stack)과 큐(Queue)는 `ArrayDeque`로 통일

자바에서는 `Stack` 클래스와 `Queue` 인터페이스(주로 `LinkedList` 구현체)를 따로 사용했지만, 코틀린에서는 이 둘의 기능을 모두 제공하는 훨씬 더 효율적이고 강력한 **`ArrayDeque<T>`** (양방향 큐)를 사용하는 것이 표준입니다.

`ArrayDeque`는 리스트의 양쪽 끝(first, last)에서 요소를 추가하거나 제거하는 작업을 O(1) 시간 복잡도로 매우 빠르게 수행합니다.

**스택 (Stack - LIFO)으로 활용하기:**
(마지막에 넣은 것을(Last-In) 먼저 꺼낸다(First-Out))

```kotlin
val stack = ArrayDeque<Int>()

// Push 연산 -> addLast()
stack.addLast(1)
stack.addLast(2)
stack.addLast(3) // Stack: [1, 2, 3]

// Pop 연산 -> removeLast()
val lastItem = stack.removeLast() // 3을 반환
println(lastItem) // 3
println(stack) // [1, 2]
```

**큐 (Queue - FIFO)로 활용하기:**
(먼저 넣은 것을(First-In) 먼저 꺼낸다(First-Out))

```kotlin
val queue = ArrayDeque<String>()

// Enqueue 연산 -> addLast()
queue.addLast("A")
queue.addLast("B")
queue.addLast("C") // Queue: [A, B, C]

// Dequeue 연산 -> removeFirst()
val firstItem = queue.removeFirst() // "A"를 반환
println(firstItem) // A
println(queue) // [B, C]
```

알고리즘 문제 풀이 시, 스택이나 큐가 필요하다면 고민 없이 `ArrayDeque`를 사용하면 됩니다.

#### 연결 리스트(Linked List)와 트리(Tree): `data class`와 널 안정성의 힘

노드 기반의 자료 구조를 만들 때, 코틀린의 `data class`와 널 가능 타입(`?`)은 완벽한 조합을 이룹니다.

```kotlin
// 연결 리스트 노드 정의
data class ListNode(
    val value: Int,
    var next: ListNode? = null // '다음 노드가 없을 수 있음'을 널 가능 타입으로 완벽하게 표현
)

// 이진 트리 노드 정의
data class TreeNode(
    val value: Int,
    var left: TreeNode? = null,
    var right: TreeNode? = null
)
```

  * **`data class`:** `equals()`, `hashCode()`, 그리고 디버깅에 매우 유용한 `toString()`을 자동으로 생성해 줍니다.
  * **`var next: ListNode?`**: 노드의 '다음' 연결이 `null`일 수 있음을 명시하여, 리스트의 끝이나 자식 노드가 없는 상태를 `NullPointerException` 걱정 없이 안전하게 다룰 수 있습니다.

-----

### 2\. 코틀린으로 표현하는 알고리즘

#### 그래프 순회: BFS (너비 우선 탐색)

BFS는 큐를 사용하는 대표적인 알고리즘입니다. 코틀린의 `ArrayDeque`와 컬렉션 라이브러리를 사용하면 매우 깔끔하게 구현할 수 있습니다.

```kotlin
// 그래프는 인접 리스트(Map) 형태로 표현
val graph = mapOf(
    1 to listOf(2, 3),
    2 to listOf(4),
    3 to listOf(4, 5),
    4 to listOf(),
    5 to listOf()
)

fun bfs(graph: Map<Int, List<Int>>, startNode: Int) {
    val visited = mutableSetOf<Int>()
    val queue = ArrayDeque<Int>() // BFS를 위한 큐

    queue.addLast(startNode)
    visited.add(startNode)

    while (queue.isNotEmpty()) {
        val node = queue.removeFirst() // 큐에서 하나를 꺼냄
        println(node)

        // 인접한 노드들을 순회
        graph[node]?.forEach { neighbor ->
            if (neighbor !in visited) { // 코틀린의 'in' 연산자 활용
                visited.add(neighbor)
                queue.addLast(neighbor) // 큐에 추가
            }
        }
    }
}
// bfs(graph, 1) 실행 시 "1, 2, 3, 4, 5" 순서로 탐색 (레벨 순)
```

#### 이진 탐색 (Binary Search)

정렬된 리스트에서 특정 값을 빠르게 찾는 이진 탐색은 알고리즘의 기본입니다. 하지만 코틀린을 사용한다면, **이것을 직접 구현할 필요가 거의 없습니다.** 표준 라이브러리가 이미 최적화된 구현을 제공하기 때문입니다.

```kotlin
val sortedList = listOf(2, 5, 8, 12, 16, 23, 38, 56)

// binarySearch()는 값을 찾으면 해당 인덱스를, 
// 찾지 못하면 값이 삽입되어야 할 위치를 (음수 값)으로 반환합니다.
val index = sortedList.binarySearch(23)

println("23의 위치: $index") // 출력: 23의 위치: 5
```

코틀린으로 알고리즘 문제를 해결한다는 것은 이처럼 언어의 간결함(`data class`, `null safety`)을 활용하여 자료 구조를 명확하게 모델링하고, `ArrayDeque`나 `binarySearch`처럼 강력한 표준 라이브러리를 활용하여 로직 자체에만 집중하는 것을 의미합니다.