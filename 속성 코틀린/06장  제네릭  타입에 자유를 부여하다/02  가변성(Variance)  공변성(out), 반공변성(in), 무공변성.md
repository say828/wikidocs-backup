### 가변성(Variance): 공변성(out), 반공변성(in), 무공변성

제네릭은 매우 강력하지만, 상속 관계와 엮이면 한 가지 미묘하고 중요한 질문에 부딪히게 됩니다. **" `Dog`가 `Animal`의 하위 타입일 때, `Box<Dog>`는 `Box<Animal>`의 하위 타입이라고 할 수 있을까?"** 즉, `Dog`를 담은 상자를 `Animal`을 담는 상자가 필요한 곳에 대신 사용할 수 있을까요?

이처럼 제네릭 타입 간의 상속 관계를 어떻게 다룰 것인지를 정의하는 개념이 바로 \*\*가변성(Variance)\*\*입니다. 코틀린에는 세 가지 종류의 가변성이 있습니다.

-----

#### 1\. 무공변성 (Invariance): 엄격한 일치 (기본값)

코틀린의 제네릭 클래스는 기본적으로 \*\*무공변(Invariant)\*\*입니다. 이는 `Dog`가 `Animal`의 하위 타입이라 할지라도, `MutableList<Dog>`와 `MutableList<Animal>`은 **서로 아무 관계가 없는 전혀 다른 타입**으로 간주됨을 의미합니다.

왜 이렇게 엄격한 제한을 두는 것일까요? 바로 **타입 안전성** 때문입니다. 만약 `MutableList<Dog>`를 `MutableList<Animal>` 타입 변수에 할당하는 것이 허용된다고 가정해 봅시다.

```kotlin
val dogs: MutableList<Dog> = mutableListOf(Dog("해피"))

// 아래 코드가 만약 허용된다면... (실제로는 컴파일 오류!)
// val animals: MutableList<Animal> = dogs 

// animals 리스트는 Animal을 담을 수 있으므로 Cat을 추가하는 것이 문법적으로 가능해진다.
// animals.add(Cat("나비"))

// 하지만 animals와 dogs는 같은 리스트 객체를 가리키고 있다.
// 결국 Dog만 들어있어야 할 dogs 리스트에서 Cat을 꺼내려 하게 되고,
// val myDog: Dog = dogs[1] // 여기서 ClassCastException 런타임 오류가 발생!
```

이처럼 타입 시스템에 구멍이 생겨 런타임 오류를 유발할 수 있기 때문에, 읽고 쓰는 것이 모두 가능한 제네릭 클래스는 기본적으로 무공변입니다.

-----

#### 2\. 공변성 (Covariance): `out` - 생산만 가능한 역할 (Producer)

만약 제네릭 클래스가 데이터를 **꺼내기만(생산, out)** 하고 **넣지는(소비, in) 못하는** 읽기 전용이라면 어떨까요? `List<Dog>`를 `List<Animal>`로 취급해도 안전합니다. 왜냐하면 `List<Animal>`에서 값을 꺼내면 `Animal`이 나올 것으로 기대하는데, 실제로는 그 하위 타입인 `Dog`가 나오므로 아무런 문제가 없기 때문입니다.

이처럼 `A`가 `B`의 하위 타입일 때 `C<A>`가 `C<B>`의 하위 타입이 되는 관계를 \*\*공변(Covariant)\*\*이라고 합니다. 코틀린에서는 타입 파라미터 앞에 **`out`** 키워드를 붙여 공변성을 표시합니다.

**규칙:** `out`으로 지정된 타입 파라미터는 해당 클래스 내에서 **반환 타입(out 위치)으로만** 사용될 수 있으며, 함수의 파라미터 타입(in 위치)으로는 사용될 수 없습니다.

코틀린의 표준 `List<E>` 인터페이스가 바로 공변으로 정의되어 있습니다.

```kotlin
// 표준 라이브러리의 List 인터페이스 (간략화)
interface List<out E> : Collection<E> {
    fun get(index: Int): E // E가 반환 타입(out 위치)에만 사용됨
    // fun add(element: E) // 이런 함수는 List 인터페이스에 존재하지 않음!
}

fun main() {
    val dogs: List<Dog> = listOf(Dog("해피"))
    val animals: List<Animal> = dogs // OK! List는 공변이므로 안전하게 할당 가능
}
```

-----

#### 3\. 반공변성 (Contravariance): `in` - 소비만 가능한 역할 (Consumer)

반대의 경우를 생각해 봅시다. 제네릭 클래스가 데이터를 **넣기만(소비, in)** 하고 **꺼내지는(생산, out) 못한다면** 어떨까요? 예를 들어, 두 객체를 비교하는 `Comparator<T>`가 있다고 가정해 봅시다. `Animal`을 비교할 수 있는 `Comparator<Animal>`는 당연히 `Dog`도 비교할 수 있습니다. 즉, `Comparator<Dog>`가 필요한 곳에 더 일반적인 `Comparator<Animal>`를 사용해도 안전합니다.

이처럼 `A`가 `B`의 하위 타입일 때 `C<B>`가 `C<A>`의 하위 타입이 되는, 상속 관계가 뒤집히는 것을 \*\*반공변(Contravariant)\*\*이라고 합니다. 코틀린에서는 타입 파라미터 앞에 **`in`** 키워드를 붙여 반공변성을 표시합니다.

**규칙:** `in`으로 지정된 타입 파라미터는 해당 클래스 내에서 **파라미터 타입(in 위치)으로만** 사용될 수 있으며, 함수의 반환 타입(out 위치)으로는 사용될 수 없습니다.

```kotlin
// 표준 라이-브러리의 Comparator 인터페이스 (간략화)
interface Comparator<in T> {
    fun compare(e1: T, e2: T): Int // T가 파라미터 타입(in 위치)에만 사용됨
}

fun main() {
    // 동물의 나이로 비교하는 비교기
    val animalComparator: Comparator<Animal> = compareBy { it.age }
    
    // Dog를 비교해야 하는 곳에 Animal 비교기를 안전하게 사용할 수 있다.
    val dogComparator: Comparator<Dog> = animalComparator // OK! Comparator는 반공변
    
    val dogs = listOf(Dog("해피", 3), Dog("바둑이", 5))
    val sortedDogs = dogs.sortedWith(dogComparator)
}
```

#### 요약: PECS (Producer Extends, Consumer Super)

이 복잡한 개념을 기억하는 데 도움이 되는 유명한 약어가 있습니다.

  * **Producer `out`**: 생산자(Producer)는 공변(`out`)이며, `List<Dog>`를 `List<Animal>`처럼 더 상위 타입(extends)으로 다룰 수 있다.
  * **Consumer `in`**: 소비자(Consumer)는 반공변(`in`)이며, `Comparator<Animal>`를 `Comparator<Dog>`처럼 더 하위 타입(super)으로 다룰 수 있다.

가변성은 제네릭 타입 시스템의 안전성을 보장하면서도 유연성을 극대화하기 위한 정교한 장치입니다. `out`과 `in`을 통해 제네릭 클래스의 데이터 흐름(생산 또는 소비)을 명확히 함으로써, 컴파일러는 타입 간의 관계를 더 정확하게 추론하고 잠재적인 오류를 막아줍니다.