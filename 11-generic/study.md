## 0. Objectives
- 제네릭 함수와 클래스 정의 방법
- 타입 소거, reified
- 선언 지점과 사용 지점 변성
- 타입 별명

## 1. 타입 인자를 받는 타입 만들기: 제네릭 파라미터
- 코틀린 컴파일러는 보통 타입과 마찬가지로 타입 인자도 추론할 수 있다
```.kt
val authors = listOf("Dmitry", "Svetlana")
// List<String> 타입
```
- 빈 리스트를 만드는 경우 타입 인자를 추론할 근거가 없기 때문에 타입 인자를 명시해야 한다
```.kt
val readers: MutableList<String> = mutableListOf()
val readers = mutableListOf<String>()
```
- 또한 코틀린에는 타입 인자가 없는 제네릭 타입인 `raw type`을 허용하지 않는다. 코틀린은 처음부터 제네릭이 도입되었기 때문에 자바와 달리 하위 호완성을 고려하지 않으므로 `raw type`을 도입하지 않았다
```.java
ArrayList aList = new ArrayList();
```
- 자바에서 위 타입의 변수를 받는경우 ArrayList<Any!> 타입이 된다

### 1-1. 제네릭 타입과 함께 동작하는 함수와 프로퍼티
- 제네릭 함수는 아래와 같이 선언할 수 있다
```.kt
fun <T> List<T>.slice(indices: IntRange): List<T>
```
- 이 함수를 호출할 때 타입 인자를 명시적으로 지정해야할 것 같지만 대부분 컴파일러가 타입 인자를 추론하므로 명시할 필요가 없다
```.kt
val letters = ('a'..'z').toList()
letters.slice<Char>(0..2)
letters.slice(10..13)
```
- 마찬가지로 확장 프로퍼티를 선언할 때도 제네릭을 사용할 수 있다
```.kt
val <T> List<T>.penultimate: T
    get() = this[size - 2]

println(listOf(1, 2, 3, 4).penultimate)
```
- 확장 프로퍼티만 제네릭을 가질 수 있으며 제네릭한 일반 프로퍼티는 말이 되지 않는다
```.kt
val <T> x: T = TODO() // compile error
```

### 1-2. 제네릭 클래스를 <> 구문을 사용해 선언한다
- 코틀린도 자바와 마찬가지로 <>를 클래스나 인터페이스 뒤에 붙이면 해당 클래스나 인터페이스를 제네릭하게 만들 수 있다
```.kt
interface List<T> {
    operator fun get(index: Int): T
}
```
- 변성은 뒤에서 설명한다
- 이런 제네릭 클래스를 확장하는 클래스를 정의하려면 기반 타입의 제네릭 파라미터에 대해 타입 인자를 지정해야 한다
- 이때 구체적인 타입을 넘길 수도 있고 파라미터로 받은 타입을 사용할 수도 있다
```.kt
class StringList: List<String> {
    override fun get(index: Int): T = TODO()
}

class ArrayList<T>: List<T> {
    override fun get(index: Int): T = TODO()
}
```
- 클래스가 자신을 타입 인자로 참조하는 것도 가능하다
```.kt
interface Comparable<T> {
    fun compareTo(other: T): Int
}

class String: Comparable<String> {
    override fun compareTo(other: String): Int = TODO()
}
```
- 아래부터는 자바와 다른 코틀린 제네릭의 특징에 대해 알아본다

### 1-3. 제네릭 클래스나 함수가 사용할 수 있는 타입 제한: 타입 파라미터 제약
- 클래스나 함수에 사용할 수 있는 타입 인자를 제한할 수 있다
- 예를들어 합을 구하는 sum 함수를 List<Int>나 List<Double>에는 사용할 수 있지만 List<String>에는 사용할 수 없다
- 어떤 타입을 제네릭 타입의 타입 파라미터에 대한 upper bound로 지정하면 그 제네릭 타입을 인스턴스화할 때 사용하는 타입 인자는 반드시 그 upper bound 이거나 그 하위 타입이어야 한다
```.kt
fun <T : Number> List<T>.sum(): T
```
- 자바의 <T extends Number>와 같은표현이다
- 아래는 두 파라미터 사이에서 더 큰 값을 찾는 제네릭 함수를 작성해보자
```.kt
fun <T: Comparable<T>> max(first: T, second: T): T {
    return if (first > second) first else second
}
fun main() {
    println(max("kotlin", "java"))
    // kotlin
}
```
- max를 비교할 수 없는 값 사이에 호출하면 컴파일 오류가 발생한다
println(max("kotlin", 42))

- 드물게 타입 파라미터에 둘 이상의 제약을 가해야 하는 경우, where를 사용할 수 있다

```.kt
fun <T> ensureTrailingPeriod(seq: T) where T: CharSequence, T: Appendable {
    if(!seq.endsWith('.')) {
        seq.append('.')
    }
}
```

### 1-4 명시적으로 타입 파라미터를 널이 될 수 없는 타입으로 표시해서 널이 될 수 있는 타입 인자 제외시키기
- 아무런 upperbound가 없는 타입 파라미터는 Any?를 upper bound로 정한 파라미터와 같다
```.kt
class Processor<T> {
    fun process(value: T) {
        value?.hashCode()
    }
}

val nullableStringProcessor = Processor<String?>()
nullableStringProcessor.process(null)
```
- 널이 될 수 없게 만드려면 Any타입을 upper bound로 지정해야 한다
```.kt
class Processor<T: Any>...
```
- 자바에서 제네릭을 사용하면서 어떤 함수에만 null이 아닌 파라미터로 제약을 걸고 싶은 경우, 아래와 같이 `@NotNull`을 사용할 수 있다
```.java
public interface JBox<T> {
    /* 널이 될 수 없는 값을 Box에 넣는다 */
    void put(@NotNull T t);

    /* 널인 경우 아무것도 하지 않음 */
    void putIfNotNull(T t);
}
```

- 이것을 코틀린으로 구현하려면 어떻게 할까? 만약 제네릭 타입을 T: Any로 제한하면 더 이상 널이 될 수 있는 값을 구현에 사용할 수 없게 되며, 이는 자바와 다른 결과를 가져온다
```.kt
class KBox<T: Any>: JBox<T> {
    override fun put(t: T) { /* ... */ }
    override fun putIfNotNull(t: T ) { /* 문제 */ }
    // 널을 파라미터로 받을 수 없게됨
}
```
- 이를 해결하기 위해 타입을 사용하는 지점에서 널이 될 수 없다고 표기하는 `T & Any` 문법이 있다
```.kt
class KBox<T>: JBox<T> {
    override fun put(t: T & Any) { /* ... */ }
    override fun putIfNotNull(t: T) { /* ... */ }
}
```

## 2. 실행 시점 제네릭스 동작: 소거된 타입 파라미터와 실체화된 타입 파라미터
- JVM의 제네릭은 타입 소거를 사용해 구현된다
- 이는 실행 시점에 제네릭 클래스의 인스턴스에 타입 인자 정보가 들어있지 않다는 뜻이다
- 코틀린에서는 `inline` 함수를 이용하여 이런 제약을 우회할 수 있다. 이것을 `reified`라고 한다

### 2-1. 실행 시점에 제네릭 클래스의 타입 정보를 찾을 때 한계: 타입 검사와 캐스팅
- 자바와 마찬가지로 코틀린 제네릭 타입 인자 정보는 런타임에 지워진다
- List<String> 객체를 만들고 그 안에 문자열을 여럿 넣더라도 실행 시점에는 그 객체를 오직 List로만 볼 수 있다
- 이런 문제로 실행 시점에 `is` 검사를 통해 타입 인자로 지정한 타입을 검사 할 수는 없다
- 예를들어 사용자 입력에 따라 List<String>이나 List<Int>를 반환하는 readNumbersOrWords라는 함수가 있다고 가정하고 is 검사를 통해 리스트의 타입 파라미터를 구분하려고 하면 컴파일이 실패한다
```.kt
fun printlist(l: List<Any>) {
    when(l) {
        is List<String> -> println("Strings: $l")
        is List<Int> -> println("Integers: $l")
    }
}
```
- 실행 시점에 어떤 값이 List인지는 알 수 있지만 그 리스트의 타입 파라미터가 무엇인지는 알 수 없다.
- 이런 타입 소거 방식은 저장해야하는 타입 정보의 크기가 줄어들어 애플리케이션의 전체 메모리 사용량이 줄어든다는 나름의 장점이 있다
- 만약 어떤 값이 Set이나 Map이 아니라 List라는 사실만을 확인하고 싶은 경우, Star Projection(*)을 사용할 수 있다
```.kt
if (value is List<*>) { ... }
```
- 뒤에서 더 설명
- 인자를 알 수 없는 제네릭 타입을 표현할때 스타 프로젝션을 사용한다는 사실만 알고 있자 (자바의 List<?>와 비슷함)
- as, as? 와 같은 타입 캐스팅에도 제네릭을 사용할 수 있다
- 하지만 정확한 타입 파라미터를 알 수 없으므로 다른 타입으로 캐스팅해도 여전히 캐스팅에 성공한다는 점을 조심해야 한다
```.kt
fun printSum(c: Collection<*>) {
    val intList = c as? List<Int> // Unchecked cast: List<*> to List라는 경고 발생
    ?: throw Exception(...)
    println(intList.sum())
}
```
- 컴파일러가 캐스팅 관련 경고를 보여주지만 컴파일에는 문제가 없다
```.kt
printSum(listOf(1, 2, 3)) // 정상 동작
printSum(setOf(1, 2, 3)) // List가 아니므로 예외 발생
```
- 하지만 잘못된 타입의 원소가 들어있는 리스트를 전달하면 실행 시점에 ClassCastException이 발생한다

```.kt
fun main() {
    printSum(listOf("a", "b", "c"))
    // ClassCastException: String cannot be cast to Number
}
```
- 하지만 컴파일 시점이 원소 타입이 알려져 있는 경우는 is 검사가 가능하다
```.kt
fun printSum(c: Collection<Int>) {
    when(c) {
        is List<Int> -> println("List sum: ${c.sum()}")
        is Sum<Int> -> println("Set sum: ${c.sum()})
    }
}

fun main() {
    printSum(listOf(1, 2, 3))
    printSum(setOf(3, 4, 5))
}
```

### 2-2 reified
- 일반적으로 타입 인자의 정보는 실행 시점에 지워지기 때문에 아래처럼 사용할 수 없다
```.kt
fun <T> isA(value: Any) = value is T // error
```
- 이 함수를 inline 함수로 만들고 타입 파라미터를 reified로 만들면 value의 타입이 T의 인스턴스인지를 실행 시점에 검사할 수 있다
```.kt
inline fun <reified T> isA(value: Any) = value is T 
```
- reified를 사용하는 간단한 예제 중 하나는 filterIsInstance이다
```.kt
fun main() {
    val items = listOf("one", 2, "three")
    println(items.filterIsInstance<String>())
}
```

- 인라인 된 함수에서만 쓸 수 있는 이유는 인라인 함수의 특징을 생각하면 된다
- 인라인 함수는 그 함수가 호출되는 모든 시점에 삽입된다. 그 말은 타입 파라미터가 정확히 무엇인지 알 수 있다는 뜻이다
- 단, 자바 코드에서는 reified + inline 을 사용하는 함수를 호출할 수 없다
- 자바에서는 inline키워드가 없기 때문에 인라인 함수도 일반 함수처럼 동작한다

### 2-3 java.lang.Class 대신 reified 사용
- 일반적으로 clazz로 표현하는 java.lang.Class를 reified를 사용하여 함수로 만들 수 있다
- 예를들어 ServiceLoader의 경우 파라미터로 java.lang.Class 를 받는다
```.kt
val serviceImpl = ServiceLoader.load(Service::class.java)

inline fun <reified T> loadService() {
    return ServiceLoader.load(T::class.java)
}
// 사용
val serviceImpl = loadService<Service>()
```

### 2-4 reified 접근자 정의
```.kt
inline val <reified T> T.canonical: String
    get() = T::class.java.canonicalName
```

### 2-5 reified의 제약
- 사용 가능한 경우
1. 타입 검사와 캐스팅(is, as 등)
2. 리플렉션 API
3. java.lang.Class 얻기
4. 다른 함수를 호출할 때 타입 인자로 사용
- 사용 불가능한 경우
1. 타입 파라미터 클래스의 인스턴스 생성
2. 타입 파라미터 클래스의 동반 객체 메서드 호출
3. reified 파라미터를 요구하는 함수를 호출하면서 reified되지 않은 타입을 타입 인자로 넘기기
4. 클래스, 프로퍼티, 인라인 함수가 아닌 함수의 타입 파라미터를 reified로 지정하기
- 이로인해 reified를 사용하려면 모든 람다가 inline 되어버린다
- 특정 상황에 따라 reified는 필요하지만 람다의 inline은 필요하지 않은 경우, noinline을 사용하기도 한다

## 3 변성
- 변성(variance)는 List<String>, List<Any> 와 같이 기저 타입이 같고 타입 인자가 다른 여러 타입에 대한 관계를 설명하는 개념이다

## 3-1 변성은 인자를 함수에 넘겨도 안전한지 판단하게 해준다
- List\<Any\> 타입의 파라미터를 받는 함수에 List\<String\>을 넘기면 안전할까? 에 대한 예시를 보자
```.kt
fun printContents(list: List<Any>) {
    println(list.joinToString())
}
fun main() {
    printContents(listOf("abc", "bac"))
}
```
- 이 경우 문자열 리스트도 정상 동작한다. 다른 예시를 보자
```.kt
fun addAnswer(list: MutableList<Any>) {
    list.add(42)
}
```
- 이 함수에 문자열 리스트를 넘기면 어떻게 될까?
```.kt
fun main() {
    val strings = mutableListOf("abc", "bac")
    addAnswer(strings) // 만약 이 줄이 컴파일된다면?
    println(strings.maxBy { it.length }) // 실행시점에 예외가 발생할 것이다
}
```
- 하지만 실제로 이 코드는 컴파일 되지 않는다.
- 그럼 List\<Any\> 타입의 파라미터를 가진 함수에 List\<String\>을 넘기면 안전한가?
- 원소 추가나 변경이 없는 경우에 한해 안전하다고 볼 수 있다
- 이처럼 여러 상황에서 단순히 읽기만 하거나 쓰기만 한다면 안정적인 관계가 분명 존재한다. 이런것들을 개발자가 허용해주도록 제어할 수 있다

### 3-2 클래스, 타입, 하위 타입
- 상속 관계를 나타내는 의미이다
- Int는 Number의 하위 타입이다
- Int는 Any의 하위 타입이다
- Int는 Int의 하위 타입이다
- Int는 String의 하위 타입이 아니다
- Int는 Int?의 하위 타입이다
- Int?는 Int의 하위 타입이 아니다
- 이처럼 타입 간 관계가 존재하는데 코틀린에서는 기본적으로 제네릭에서는 이런 관계가 허용되지 않는다
- 예를들어 List\<Int\>는 List\<Any\>의 하위 타입이 아니다
- List\<Int\>는 List\<Int?\>의 하위 타입이 아니다
- 이처럼 어떤 제네릭 타입이 있는데 서로 다른 두 타입 A, B에 대해 서로 하위타입도 아니고 상위타입도 아닌 경우 제네릭 타입이 타입 파라미터에 대해 무공변(invariant) 라고 말한다
- 자바에서도 기본적으로 모든 클래스가 무공변이다
- 읽기전용 collection인 List를 기준으로 아래에서 A가 B의 하위 타입이면 List\<A\>도 List\<B\>임을 명시할 수 있는 공변성에 대해 알아본다

### 3-3 공변성은 하위 타입 관계를 유지한다
- 공변적인 클래스는 A가 B의 하위타입일때 Producer\<A\>가 Producer\<B\>의 하위 타입인 경우를 말한다
- 이를 명시적으로 나타내기 위해서는 `out` 키워드를 사용한다
```.kt
interface Producer<out T> {
    fun produce(): T
}
```
- 클래스의 타입 파라미터를 공변적으로 만들면 함수 정의에 사용한 파라미터의 타입과 타입 인자의 타입이 정확하게 일치하지 않아도 그 클래스의 인스턴스를 함수의 인자나 반환값으로 사용될 수 있다

- 대표적으로 List의 구현을 보면 out 키워드를 사용하고 있다
```.kt
public actual interface List<out E> : Collection<E> {
    ...
}
```
- 따라서 우리가 사용하는 List\<String\>은 List\<Any\>의 하위 타입이고 List\<Int\>는 List\<Number\>의 하위타입이 되는 것이다.
- 하지만 모든 클래스를 공변적으로 만들 수 는 없다
- 공변적으로 만들면 안전하지 못한 클래스들이 존재한다
- 따라서 타입 파라미터를 공변적으로 지정하면 클래스 내부에서 그 파라미터를 사용하는 방법을 제한한다
- 타입 안정성을 보장하기 위해 공변적 파라미터는 항상 아웃(out) 위치 에만 있어야 한다

```.kt
interface Transformer<T> {
    fun transform(t: T)   :   T
                    ---      ---
                   in위치    out위치
}
```

- 요약하면 out을 사용하면 하위 타입 관계가 유지된다
- 또한 T를 아웃 위치에서만 사용할 수 있다
- List를 사용하는 경우에도 항상 아웃 위치에만 T가 쓰이는것을 볼 수 있다
```.kt
interface List<out T> : Collection<T> {
    operator fun get(index: Int): T // out 위치

    fun subList(fromIndex: Int, toIndex: Int): List<T>
}
```
- 다만 이 책에서 어떤 위치가 아웃이고 인인지 판정하는 정확한 알고리즘은 다루지 않는다 코틀린 언어 문서 참고
- 예외적으로 생성자 파라미터는 인, 아웃 그 어떤 위치도 아니다
- 타입 파라미터가 out이라 하더라도 그 타입을 여전히 생성자 파라미터 선언에 사용할 수 있다
```.kt
class Herd<out T: Animal>(vararg animals: T) { /* ... */ }
```
- 변성의 역할은 코드에서 위험할 여지가 있는 메서드를 호출할 수 없게 만드는 역할이므로 생성자는 위험할 여지가 없다
- 하지만 val, var 키워드를 사용하는 경우 게터, 세터 메서드를 정의하는 것과 같이 때문에 
- 읽기 전용 프로퍼티인 val는 아웃, var는 아웃과 인 위치 모두 해당한다
```.kt
class Herd<T: Animal>(var leadAnimal: T, vararg animals: T) { /* ... */ }
```
- 여기서 T타입은 프로퍼티가 var(인, 아웃 모두 해당) 인 위치에 있기 때문에 T를 out으로 표기할 수 없다
- 이런 변성 규칙은 오직 외부에서 볼 수 있는 클래스에만 적용되므로, leadAnimal을 private으로 선언한 경우에는 out을 사용할 수 있다
```.kt
class Herd<out T: Animal>(private var leadAnimal: T, vararg animals: T) { /* ... */ }
```

### 3-4 반공변성(contravariance)는 하위 타입 관계를 뒤집는다
- 반공변성은 거울에 비친 상이라 할 수 있다
- 반공변 클래스의 하위 타입 관계는 그 클래스의 타입 파라미터의 상하위 타입 관계와 반대다
- 반공변성은 in 키워를 사용한다
```.kt
interface Comparator<in T> {
    fun compare(e1: T, e2: T): Int { /* ... */ }
} // T를 in 위치에 사용
``` 
- 즉 Comparator<Any>가 Comparator<String>의 하위 타입이라는 뜻이다
- 기존의 String이 Any의 하위타입인것과는 정반대 방향이다
- 즉 타입 B가 타입 A의 하위 타입일 때 Consumer<A>가 Consumer<B>의 하위 타입인 관계가 성립하면 제네릭 클래스는 타입 인자 T에 대해 반공변이다
- in이라는 키워드는 그 키워드가 붙은 타입이 이 클래스의 메서드 안으로 전달돼 메서드에 의해 소비된다는 뜻이다
- 공변성의 경우와 마찬가지로 타입 파라미터의 사용을 제한함으로써 특정 하위 타입 관계에 도달할 수 있다
- 클래스나 인터페이스가 어떤 타입 파라미터에 대해 공변적이면서 다른 타입 파라미터에 대해서는 반공변적일 수도 있다
```.kt
interface Function1<in P, out R> {
    operator fun invoke(p: P): R
}
```
- Function1이 (P) -> R 과 같은 함수 타입이라는 것은 앞에서 배웠다
- 그렇다면 아래와 같은 상황이 성립한다
- Animal 인터페이스를 상속받는 Cat 클래스가 있다고 가정하면
- (Cat) -> Number 는 (Animal) -> Int 의 하위 타입이다
- 파라미터는 out, 리턴타입은 in이기 때문이다

### 3-5 사용 지점 변성을 사용해 타입이 언급되는 지점에서 변성 지정
- 클래스를 선언하면서 변성을 지정하면 그 클래스를 사용하는 모든 장소에 변성이 영향을 받으므로 편리하다
- 이런 방식을 선언 지점 변성(declaration site variance)라고 부른다
- 하지만 어떤 클래스 안에서 변성을 지정할 수 없는 경우, 특정 파라미터가 나타 나는 지점에서 변성을 정할 수 있다
- MutableList와 같은 많은 인터페이스는 공변적이지도 반공변적이지도 않다
- 하지만 그런 타입의 변수가 한 함수 안에서 생산자나 소비자 중 단 한 가지 역할만을 담당하는 경우가 자주 있다
- 아래 예시를 보자
```.kt
fun <T> copyData(source: MutableList<T>, destination: MutableList<T>) {
    for(item in source) {
        destination.add(item)
    }
}
```
- 컬렉션의 원소를 다른 컬렉션으로 복사하는 함수이다
- 원본 컬렉션의 경우 읽기만 하고 대상 컬렉션에는 쓰기만 하기 때문에 원소의 타입이 정확히 일치할 필요는 없다
- 예를들어 문자열이 원소인 컬렉션에서 Any 컬렉션으로 원소를 복사해도 아무 문제가 없다
- 이를 해결하기 위해서 두 번째 제네릭 파라미터를 쓸 수 있다
```.kt
fun <T: R, R> copyData(source: MutableList<T>, destination<R>) {
    ...
}
```
- 이를 변성을 사용하여 좀 더 다듬을 수 있다
```.kt
fun <T> copyData(source: MutableList<out T>, destination: MutableList<T>) {
    ...
}
```
```.kt
fun main() {
    val list: MutableList<out Number> = mutableListOf()
    list.add(42) // T인 42가 타입 파라미터에 직접 사용, in 위치에 사용되었으므로 compile error
}
```
- List\<out T\> 처럼 out이 이미 지정된 타입 파라미터를 out 프로젝션 하는 것은 의미 없다
- List의 정의는 이미 class List\<out T\>이므로 List\<out T\>는 그냥 List\<T\>와 같다
- 컴파일러는 불필요한 프로젝션이라는 경고를 보낸다

### 3-6 스타 프로젝션
- 제네릭 타입 인자에 대한 정보가 없음을 표현할 때 *을 사용하며 List\<*\>와 같이 나타낸다
- MutableList\<*\>는 MutableList\<Any?\>와 다르다 (MutableList는 T에 대해 무공변성)
- MutableList\<Any?\>는 어떤 타입이든 다 담을 수 있다는 의미이지만 MutableList\<*\>는 어떤 타입인지 정확히 모르지만 한번 정해지면 그 타입이 되는 것이다
- MutableList\<*\>와 같이 리스트를 선언할 수는 없다
- MutableList\<*\>로부터 원소를 얻을 수는 있다
- 진짜 타입은 알 수 없지만 어쨌든 그 원소 타입이 Any?의 하위 타입이라는 것은 분명하다
- 일반적으로 타입 인자에 대한 정보가 중요하지 않을 때 스타 프로젝션을 사용한다
```.kt
fun printFirst(list: List<*>) {
    if(list.isNotEmpty()) {
        println(list.first()) // Any? 타입 반환
    }
}

fun main() {
    printFirst(listOf("Sveta", "Seb", "Dima", "Roman"))
}
```
- 제네릭을 쓰면 정확한 파라미터 타입을 받을 수 있다
```.kt
fun <T> printFirst(list: List<T>) {
    if(list.isNotEmpty()) {
        println(list.first())
    }
}
```
- 스타 프로젝션이 더 간결하지만 제네릭 타입이 어떤 타입인지 알 필요가 없는 경우에만 스타 프로젝션을 쓰는것이 좋다

### 3-7 typealias
- 어떤 특정 타입에 명확한 이름을 지어주고 싶은 경우 typealias를 사용한다
- 예를들어 (String, String) -> String 이라는 람다 타입에 이름을 붙일 수 있다
```.kt
typealias NameCombiner = (String, String) -> String

val bandCombiner: NameCombiner = { a, b -> ... }
```
- typealias를 사용하면 읽을 때 좀 더 쉽게 이해가 되도록 할 수 있다
- 하지만 이는 가칭일 뿐 컴파일 타임에 원래의 타입으로 돌아간다
- 코틀린에는 primitive type 하나에 특정 의미를 주고 싶은 경우 value class(구 inline class)를 사용하는 방법도 있다
```.kt
typealias ValidatedInput = String

@JvmInline
value class ValidatedInput(val s: String)
```
- typealias는 컴파일 시점에 아무것도 검증해 주지 않는다
- 현재 예시로 사용한 ValidatedInput은 String 타입에도 사용할 수 있고, String타입에 ValidatedInput을 사용할 수도 있다
- 따라서 최소한의 부가 비용으로 타입 안정성을 추가하는 것이 목적이라면 value class를 사용하는것이 더 좋다

### 부록 crossinline
- 코틀린에서는 non-local return 이 허용되지 않는다
- 아래 예시를 보자
```.kt
fun foo() {
    ordinaryFunction {
        return // ERROR label이 필요함
    }
}
```
- 그러나 만약 람다함수라면, 함수가 본문으로 인라이닝 되므로 return이 가능하다
```.kt
fun foo() {
    inlined {
        return // OK
    }
}
```
- 이렇게 함수 내에 존재하는 다른 람다 블록에서 return 하는것을 non-local 리턴이라고 부른다
- 그렇다면 inline 함수의 파라미터로 전달된 람다를 다른 람다 블록에서 사용할 수 있을까?
```.kt
inline fun f(body: () -> Unit) {
    val f = object: Runnable {
        override fun run() = body() //
    }
}
// Can't inline 'body' here: it may contain non-local returns. Add 'crossinline' modifier to parameter declaration 'body'
```
- 이처럼 다른 람다에 함수를 전달하는 경우 crossinline 키워드를 사용해야 한다
- 즉, 내 범위를 벗어나는 곳에서 람다가 어떤 영향을 줄지 알수 없기 때문에 사용할 수 없도록 만들었다
```.kt
fun func() {
    secondaryFunc {
        println("ddddd")
        return
    }
}

inline fun secondaryFunction(lambda: () -> Unit) {
    view.setOnClickListener {
        lambda()
    }
}
```


