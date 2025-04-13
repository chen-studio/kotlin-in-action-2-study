## 0.Objectives 
- 연산자 오버로딩에 대한 이해
- 여러 연산을 지원하기 위해 특별한 이름이 붙은 메서드
- 위임 프로퍼티

## 1. operator fun (산술 연산자)
- 코틀린에서 미리 정해진 이름의 함수가 일부 존재한다
- 이런 미리 정해진 이름의 함수를 연결해 주는 기법을 관례 라고 부른다
- 아래 Point 객체를 기준으로 설명한다
```.kt
data class Point(val x: Int, val y: Int)
```

### 1-1. plus, times, divde 등: 이항 산술 연산 오버로딩
- 두 점을 더하는 연산을 구현해보자
```.kt
data class Point(val x: Int, val y: Int) {
    operator fun plus(other: Point): Point =
        Point(x + other.x, y + other.y)
}

fun main() {
    val p1 = Point(10, 20)
    val p2 = Point(30, 40)
    println(p1 + p2) // Point(x=40, y=60)
}
```
- plus 앞에 operator가 붙은 것처럼 연산자 오버로딩을 사용할땐 operator 키워드가 있어야 한다
- 연산자를 확장함수로 정의할 수도 있다
```.kt
operator fun Point.plus(other: Point): Point =
    Point(x + other.x, y + other.y)
```
- 코틀린에서는 프로그래머가 직접 연산자를 만들어 사용할 수는 없고 언어에서 미리 정해둔 연ㄴ산자만 오버로딩할 수 있다
- a * b / times
- a / b / div
- a % b / mod
- a + b / plus
- a - b / minus
- 직접 정의한 함수를 통해 구현하더라도 연산자 우선순위는 언제나 표준 숫자 타입에 대한 연산자 우선순위와 같다
- *, /, % > +, -
- 코틀린에서 자바를 호출하는 경우, 함수 이름이 코틀린의 관례에 맞아 떨어지기만 하면 항상 연산자 식을 사용해 그 함수를 호출할 수 있다
- 자바에서는 따로 연산자에 표시를 할 수 없으므로 operator 없이 이름과 파라미터 개수만 고려하면 된다
- 자바 메서드가 이미 있다면 관례에 맞는 이름을 가진 확장함수를 작성하고 연산을 기존 자바 메서드에 위임하면 된다

- 연산을 정의할때 두 피연산자가 같은 타입일 필요는 없다
```.kt
data class Point(val x: Int, val y: Int) {
    operator fun times(scale: Double): Point =
        Point((x * scale).toInt(), (y * scale).toInt())
}

p * 1.5
// 
```
- 코틀린 연산자는 자동으로 교환법칙을 지원하지는 않는다
- 따라서 p * 1.5 이외에 1.5 * p로도 쓸 수 있어야 한다면 
- 같은 순서의 식에 대응하는 연산자 함수인 operator fun Double.times(p: Point): Point를 더 정의해야 한다
- 연산자 함수의 반환 타입이 꼭 두 피연산자 중 하나와 일치해야만 하는 것은 아니다
```.kt
operator fun Char.times(count: Int): String =
    toString().repeat(count)

'a' * 3 // aaa
```

- 코틀린은 표준 숫자 타입에 대해 비트 연산자를 정의하지 않는다
- 따라서 커스텀 타입에서 비트 연산자를 정의할 수도 없다
- 대신 중위 연산자 표기법을 지원하는 일반 함수를 사용해 비트 연산을 수행한다
- sh1: 왼쪽 시프트 (자바 <<)
- shr: 오른쪽 시프트 (부호 비트 유지, 자바>>)
- ushr: 오른쪽 시프트 (0으로 부호 비트 설정, 자바 >>>)
- and: 비트 곱 (자바 &)
- or: 비트 합(자바 |)
- xor: 비트 배타 합(자바 ^)
- inv: 비트 반전(자바 ~)
```.kt
0x0F and 0xF0 // 0
0x0F or 0xF0 // 255
0x1 shl 4 // 16
```

### 1-2 복합 대입 연산자 오버로딩
- 연산자 오버로딩을 사용하면 + 뿐 아니라 그와 관련있는 연산자인 +=도 자동으로 함께 지원한다
= +=, -= 등의 연산자를 compound assignment(복합 대입) 연산자 라고 부른다
```.kt
var point = Point(1, 2)
point += Point(3, 4) // point = point + Point(3, 4) 와 같음
```
- 경우에 따라 += 연산이 객체에 대한 참조를 다른 참조로 바꾸기보다
- 원래 객체의 내부 상태를 변경하고 싶은 경우
```.kt
val numbers = mutableListOf<Int>()
numbers += 42
```
- 반환 타입이 Unit인 plusAssign 함수를 정의하면서 operator로 표시하면 += 연산자에 해당 함수를 사용한다
- 다른 복합 연산자도 minusAssign, timesAssign 등의 이름을 사용한다
- 코틀린 표준 라이브러리에 아래 함수가 정의되어 있다
```.kt
operator fun <T> MutableCollection<T>.plusAssign(element: T) {
    this.add(element)
}
```
- 이론적으로는 +=를 plus와 plusAssign 양쪽으로 컴파일 할 수 있다
- 다만 둘다 정의되어 있고 둘다 +=에 사용 가능한 경우 컴파일 에러가 발생한다
- 이를 해결하는 방법은 일반 함수 호출을 사용하는 것이며
- var를 val로 바꿔 plusAssign 적용이 불가능하게 하는 방법도 있다
- 결과적으로는 plus, plusAssign을 동시에 정의하지 않는 것이 좋다
- 클래스가 Point처럼 변경 불가능하다면 plus같은 새로운 값을 만환하는 연산만을 추가해야 한다
- 코틀린 표준 라이브러리는 컬렉션에 대해 두가지 연산을 제공한다
- +, -는 항상 새로운 컬렉션을 반환하며 +=, -=은 mutable collection을 대상으로 객체 상태를 변화시킨다
```.kt
val list = mutableListOf(1, 2)
list += 3 // list를 변경하여 3을 추가한다
val newList = list + listOf(4, 5) // 새로운 리스트를 반환한다
```

### 1-3 피연산자가 1개뿐인 연산자: 단항 연산자 오버로딩
- 단항 연산자를 오버로딩 하는 경우에도 동일하다
```.kt
operator fun Point.unaryMinus(): Point = Point(-x, -y)

fun main() {
    val p = Point(10, 20)
    println(-p) // Point(x=-10, y=-20)
}
```
- +a / unaryPlus
- -a / unaryMinus
- !a / not
- ++a, a++ / inc
- --a, a-- / dec
- BigDecimal 예시
```.kt
operator fun BigDecimal.inc() = this + BigDecimal.ONE

fun main() {
    var bd = BigDecimal.ZERO
    println(bd++) // 0
    println(bd) // 1
    println(++bd) // 2
}
```

## 2. 비교 연산자 오버로딩
- 자바에서 equals나 compareTo를 호출해야 하는 자바와 달리 코틀린에서는 `==` 비교 연산을 통해 비교를 사용할 수 있어 훨씬 간결하다

### 2-1 동등성 연산자 equals
- 코틀린 `==` 는 자바의 `equals`와 같다
- `!=`을 사용하는 식도 `equals`로 컴파일된다
- 다만 코틀린은 null이 아닌 경우에만 equals를 호출한다
- kotlin `a==b` => java `a?.equals(b) ?: (b == null)
- 위 예시의 Point 클래스의 경우 data class 이므로 컴파일러가 equals를 자동으로 구현해준다
```.kt
class Point(val x: Int, val y: Int) {
  override fun equals(obj: Any?): Boolean {
    if(obj === this) return true
    if(obj !is Point) return false
    return obj.x == x && obj.y == y // 스마트 캐스트 후 x와 y 프로퍼티에 접근
  }
}

Point(10, 20) == Point(10, 20) // true
Point(10, 20) == Point(5, 5) // false
null == Point(1, 2) false
```
- kotlin의 `===`은 java의 `==` 과 같다 (메모리 비교)
- `===` 은 오버로딩 할 수 없다
- equals는 최상위 타입인 Any에 정의된 메서드이므로 operator가 아닌 override를 사용한다

### 2-2 순서 연산자: compareTo(<, >, <=, >=)
- 자바에서 값을 비교할때 Comparable 인터페이스를 상속받아 compareTo 메서드를 구현한다
- 자바에서는 이 메서드를 짧게 호출할 방법이 없지만 코틀린은 부등호를 사용하면 자바에서 compareTo 메서드로 컴파일 된다
- a >= b ==> a.compareTo(b) >= 0
- 지금까지 사용했던 Point클래스는 2차원이므로 비교연산이 명확하지 않으므로 Person 클래스를 사용해보자
```.kt
class Person(
  val firstName: String, val lastName: String
) : Comparable<Person> {
  override un compareTo(other: Person): Int {
    return compareValuesBy(this, other, Person::lastName, Person::firstName)
  }
}
```
- compareValuesBy는 아래와 같이 구현되어 있다
```.kt
public fun <T> compareValuesBy(a: T, b: T, vararg selectors: (T) -> Comparable<*>?): Int {
    require(selectors.size > 0)
    return compareValuesByImpl(a, b, selectors)
}

private fun <T> compareValuesByImpl(a: T, b: T, selectors: Array<out (T) -> Comparable<*>?>): Int {
    for (fn in selectors) {
        val v1 = fn(a)
        val v2 = fn(b)
        val diff = compareValues(v1, v2)
        if (diff != 0) return diff
    }
    return 0
}
```
- 인자로 받은 두 객체를 비교하고 selectors 연산에 따라 두 객체의 비교 대상이 되는 프로퍼티의 비교 연산이 0이 아닌값이 나올때 까지 반복한다
- 위 예시의 경우 lastName을 비교하고 lastName이 같으면 firstName을 비교하여 결과를 반환한다
- compareTo도 Comparable 인터페이스의 메서드이므로 operator가 아닌 override를 사용한다

## 3. 컬렉션과 범위에 대해 쓸 수 있는 관례
- 컬렉션에서 사용할 수 있는 여러 관례에 대해 알아본다

### 3-1 get, set
- 코틀린에서 맵에 접근할 때 자바의 배열에 접근할때 처럼 각 괄호를 사용한다(`[]`)
```.kt
val value = map[key]
mutableMap[key] = newValue
```
- 이런 동작이 가능한 이유는 Map과 MutableMap에 get과 set이 연산자 오버로딩 되어있기 때문이다
- 이런 동작을 Point 객체로 직접 구현해보자
```.kt
operator fun Point.get(index: Int): Int {
  return when(index {
    0 -> x
    1 -> y
    else -> throw ...
  }
}

fun main() {
  val p = Point(10, 20)
  println(p[1]) // 20
}
```
- get의 파라미터로 Int가 아닌 타입도 사용할 수 있으며 2차원 행렬 객체를 생각해보면 get의 파라미터로 여러개의 파라미터도 사용할 수 있다
- set 연산자도 살펴보자
```.kt
data class MutablePoint(var x: Int, var y: Int)

operator fun MutablePoint.set(index: Int, value: Int) {
    when(index) {
        0 -> x = value
        1 -> y = value
        else -> throw ...
    }
}

val p = MutablePoint(10, 20)
p[1] = 42
println(p) // MutablePoint(x=10, y=42)
```

### 3-2 in
- 어떤 객체가 컬렉션에 들어가 있는지를 확인하려면 in을 사용한다
- 이에 대응하는 연산 함수는 contains이다
```.kt
data class Rectangle(val upperLeft: Pont, val lowerRight): Point)

operator fun Rectangle.contains(p: Point): Boolean {
    return p.x in upperLeft.x..<lowerRight.x &&
        p.y in upperLeft.y..<lowerRight.y
}
```
- (개인) ..(닫힌 범위) 과 ..<(열린 범위) 가 있는데, ..<는 거의 사용해본적이 없고 until을 많이 사용했음 (1 until 3)
- a in c ==> c.contains(a)
- 10..20 은 10<=target<=20
- 10 until 20 (또는 10..<20) 은 10<=target<20

### 3-3 객체로부터 범위 만들기: rangeTo, rangeUntil
- start..end 는 start.rangeTo(end) 로 컴파일 되나
- 이 연산자도 새로 정의할 수 있지만 Comprable 인터페이스에는 rangeTo가 이미 정의되어 있다
```.kt
operator fun<T: Comparable<T>> T.rangeTo(that: T): ClosedRange<T>
```
- rangeTo는 함수의 범위를 반환하며 어떤 원소가 그 범위 안에 들어있는지 검사할 수 있게 해준다
- 다만 0..n.forEach {} 는 범위 연산자의 우선순위가 낮아 컴파일되지 않으므로 괄호를 포함해야 한다
- (0..n).forEach {}

### 3-4 자신의 타입에 대해 루프 수행: iterator
- 코틀린의 for 루프는 in 연산자를 사용한다 `for(x in list) { ... }
- 내부적으로는 list.iterator()를 호출해서 자바와 마찬가지로 hasNext와 next 호출을 반복한다
- 하지만 이 또한 관례이므로 iterator 메서드를 확장함수로 정의할 수 있으며 이런 성질로 인해 자바 문자열에 대한 for 루프가 가능하다
- 코틀린 표준 라이브러리는 String의 상위 클래스인 CharSequence에 대한 iterator 확장함수를 제공한다
```.kt
operator fun CharSequence.iterator(): CharIterator
for(c in "abc") { }
```
- 마찬가지로 클래스 안에 itrator 메서드를 구현하거나 확장함수로 만들 수 있다
```.kt
operator fun ClosedRange<LocalDate>.iterator(): Iterator<LocalDate> =
    object: Iterator<LocalDate> {
        var current = start
        override fun hasNext() = current <= endInclusive

        override fun next(): LocalDate {
            val thisDate = current
            current = current.plusDays(1)
            return thisDate
        }
    }
```

## 4. component 함수를 사용해 구조 분해 선언 제공
```.kt
val p = Point(10, 20)
val (x, y) = p
```
- 일반 선언과 비슷해보지만 =의 왼쪽을 괄호로 묶었다
- 이것을 destructing declaration(구조 분해 선언) 이라고 한다
- 내부적으로 componentN 이라는 함수를 호출한다
- `val (a, b) = p` => `val a = p.component1()` `val b = p.component2()`
- data class의 경우 생성자의 프로퍼티에 한해 컴파일러가 componentN 함수를 구현해 준다
- 일반 클래스에서 아래와 같이 직접 구현할 수 있다
```.kt
class Point(val x: Int, val y: Int) {
    operator fun component1() = x
    operator fun component2() = y
}
```
- 무한히 componentN을 선언할 수 없지만 destructing declaration은 유용하게 사용할 수 있다
- 코틀린 표준 라이브러리에서 컬렉션의 맨 앞의 다섯 원소에 대한 componentN을 제공한다
- 컬렉션 예시
```.kt
val (n, m) = readLine().split(" ") // 최대 5개까지 가능
```
- 가장 단순한 방법은 Pair, Triple 클래스를 사용하는 것이지만 그 안에 담겨있는 의미를 알 수 없다는 단점이 있다

### 4-1 구조 분해 선언과 루프
- 함수 본문 내 선언문뿐 아니라 변수 선언이 들어갈 수 장소라면 어디든 사용할 수  ㅣㅆ다
```.kt
fun printEntries(map: Map<String, String>) {
    for((key, value) in map) {
        println("$key -> $value")
    }
}

fun main() {
    val map = mapOf("Oracle" to "Java", "JetBrains" to "Kotlin")
    printEntries(map)
}
```
- 실제 코드는 아래와 같이 컴파일된다
```.kt
for (entry in map.entries) {
    val key = entry.component1()
    val value = entry.component2()
    ...
}
```
- 람다가 data class나 맵 같은 복합적인 값을 파라미터로 받을 때도 구조 분해 선언을 쓸 수 있다
```.kt
map.forEach { (key, value) ->
    println("$key -> $value")
}
```

### 4-2 문자를 사용해 구조 분해 값 무시
```.kt
data class Person(
    val firstName: String,
    val lastName: String,
    val age: Int,
    val city: String
)
```
- city(마지막 프로퍼티)를 제외한 3개의 프로퍼티만 필요한 경우
```.kt
val (firstName, lastName, age) = p
```
- 중간 프로퍼티가 필요 없는 경우
```.kt
val (firstName, _, age) = p
```
- 구조분해는 순서에 따라 동작하기 때문에 변수명을 잘 지정해야 한다
```.kt
val (lastName, firstName, age, city) = p // lastName과 firstName이 잘못 지정됨
```

## 5. 프로퍼티 접근자 로직 재활용: 위임 프로퍼티 (by delegate 패턴)
- 위임은 객체가 직접 작업을 수행하지 않고 다른 도우미 객체가 그 작업을 처리하도록 맡기는 디자인 패턴을 말한다
- 이때 작업을 처리하는 도우미 객체를 delegate object(위임 객체)라고 한다

### 5-1 위임 프로퍼티의 기본 문법과 내부 동작
- 기본적인 문법은 아래와 같음
```.kt
var p: Type by Delegate()
```
- p 프로퍼티는 접근자 로직을 다른 객체에 위임한다
- 컴파일러는 숨겨진 도우미 프로퍼티를 만들고 그 프로퍼티를 위임 객체의 인스턴스로 초기화한다.
```.kt
class Foo {
    var p: Type by Delegate()
}

class Foo {
    private val delegate = Delegate()

    var p: Type
        set(value: Type) = delegate.setValue(/* ... */, value) 
        get() = delegate.getValue(/* ... */)
}
```
- 프로퍼티 위임 관례에 따라 Delegate 클래스는 getValue와 setValue 메서드를 제공해야 한다
- 또한 객체 최초 생성시 검증 로직을 수행하거나 위임이 인스턴스화 되는 방식을 제어하려면 선택적으로 `provideDelegate` 함수를 구현할 수 있다
```.kt
class Delegate {
    operator fun getValue(...) { ... }
    operator fun setValue(..., value: Type) { ... }
    operator fun provideDelegate(...): Delegate {...} // 위임 객체 생성 또는 제공하는 로직 담당
}

class Foo {
    var p: Type by Delegate()
}

fun main() {
    val foo = Foo() // 위임 프로퍼티는 있는 타입의 객체를 생성하는데, 위임 객체에 provideDelegate가 있으면 그 함수를 호출해 위임 객체를 생성
    val oldValue = foo.p // delegate.getValue 호출
    foo.p = newValue // delegate.setValue 호출
}
```

### 5-2 위임 프로퍼티 사용: by lazy()를 사용한 지연 초기화
- 지연 초기화는 객체의 일부분을 초기화 하지  않고 남겨뒀다가 실제로 그 부분의 값이 필요한 경우 초기화할때 쓰이는 패턴이다
- 예시로 Email이라는 클래스가 있고 데이터베이스에서 이메일을 가져오는 loadEmail함수가 있다
```.kt
class Email { ... }
fun loadEmails(person: Person): List<Email> {
    println("${person.name}의 이메일을 가져옴")
    return listOf(...)
}
```
- 이메일을 불러오기 전에는 null을 저장하고 불러온 다음에는 이메일 리스트를 저장하는 _emails 프로퍼티를  추가헤서 지연 초기화를 직접 구현한 예시를 보자
```.kt
class Person(val name: String) {
    private var _emails: List<Email>? = null

    val emails: List<Email>
        get() {
            if (_emails == null) {
                _emials = loadEmails(this)
            }
            return _emails!!    
        }
}
```
- 여기서 backing property 라는 기법을 사용한다. 변경 가능한 프로퍼티는 _를 포함하고 포함하지 않는 프로퍼티는 읽기 전용 프로퍼티로 사용한다
- 하지만 위 코드는 쓰레드 동기화에 안전하지 않으며 매번 이렇게 구현하는 것은 매우 번거롭다
- by lazy를 사용하면 쉽게 구현할 수 있다
```.kt
class Person(val name: String) {
    val emails by lazy { loadEmails(this) }
}
```
- lazy 함수는 코틀린 관례에 맞는 시그니처의 getValue 메서드가 들어있는 객체를 반환한다.
- 참고로 lazy는 아래와 같이 구성됨
```.kt
public actual fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
    when (mode) {
        LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
        LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
        LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
    }

public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)

private class SynchronizedLazyImpl<out T>(initializer: () -> T, lock: Any? = null) : Lazy<T>, Serializable {
    private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED_VALUE
    // final field is required to enable safe publication of constructed instance
    private val lock = lock ?: this

    override val value: T
        get() {
            val _v1 = _value
            if (_v1 !== UNINITIALIZED_VALUE) {
                @Suppress("UNCHECKED_CAST")
                return _v1 as T
            }

            return synchronized(lock) {
                val _v2 = _value
                if (_v2 !== UNINITIALIZED_VALUE) {
                    @Suppress("UNCHECKED_CAST") (_v2 as T)
                } else {
                    val typedValue = initializer!!()
                    _value = typedValue
                    initializer = null
                    typedValue
                }
            }
        }
```

### 5-2 위임 프로퍼티 구현
- 위임 프로퍼티를 구현하기 위한 예제
- 객체의 프로퍼티가 변경될때마다 리스너에게 변경 통지를 하고 싶은 상황
- 이런 경우를 Observable 이라고 한다
```.kt
fun interface Observer {
    fun onChanged(name: String, oldValue: Any?, newValue: Any?)
}

open class Observable {
    val observers = mutableListOf<Observer>()
    fun notifyObservers(propName: string, oldValue: Any?, newValue: Any?) {
        for(obs in observers) {
            obs.onChange(propName, oldValue, newValue)
        }
    }
}

class Person(val name: String, age: Int, salary: Int): Observable() {
    var age: Int = age
        set(newValue) {
            val oldValue = field
            field = newValue
            notifyObservers(
                "age", oldValue, newValue
            )
        }

    var salary: Int = salary
        set(newValue) {
            val oldValue = field
            field = newValue
            notifyObservers(
                "salary", oldValue, newValue
            )
        }
}

fun main() {
    val p = Person("Seb", 28, 1000)
    p.observers += Observer { propName, oldValue, newValue ->
        println(
            """    
            Property $propName changed from $oldValue to $newValue
            """
        )
    }
    p.age = 29
    //  Property age changed from 28 to 29!
    p.salary = 1500
    // Property salary changed from 1000 to 1500!
}
```
- property에 대한 반복되는 get, set 메서드를 ObservableProperty라는 클래스를 정의하여 중복을 제거해보자
```.kt
class ObservableProperty(
    val propName: String,
    var propValue: Int,
    val observable: Observable
) {
    fun getValue(): Int = propName
    fun setValue(newValue: Int) {
        val oldValue = propValue
        propValue = newValue
        observable.notifyObservers(propName, oldValue, newValue)
    }
}

class Person(val name: String, age: Int, salary: Int): Observable() {
    val _age = ObservableProperty("age", age, this)
    var age: Int
        get() = _age.getValue()
        set(newValue) {
            _age.setValue(newValue)
        }

    val _salary = ObservableProperty("salary", salary, this)
    var salary: Int
        get() = _salary.getValue()
        set(newValue) {
            _salary.setValue(newValue)
        }
}
```

- 코틀린의 위임이 실제로 작동하는 방식과 비슷하게 구현한 예시이다.
- 값이 바뀌면 자동으로 옵저버에게 변경 통지를 해주고 있다
- 이제 ObservableProperty를 delegate 패턴을 사용하여 구현해보자.
```.kt
imort kotlin.reflect.KProperty

class ObservableProperty(
    var propValue: Int,
    val observable: Observable
) {
    operator fun getValue(thisRef: Any?, prop: KProperty<*>): Int = propValue
    operator fun setValue(thisRef: Any?, prop: KProperty<*>, newValue: Int) {
        val oldValue = propValue
        propValue = newValue
        observable.notifyObservers(prop.name, oldValue, newValue
    }
}
```
- 관례에 따라 getValue, setValue에 operator가 붙는다
- 두 함수 getValue, setValue는 파라미터를 2개 받는다
- 설정하거나 읽을 프로퍼티를 나타내는 인스턴스(thisRef)와 Kotlin Reflection Property인 KProperty를 받는다
- 지금은 KProperty 객체를 통해 property의 name을 가져올 수 있다는 점만 기억하자
- KProperty를 통해 프로퍼티 이름을 전달받으므로 생성자에서 propName을 없앴다
```.kt
class Person(val name: String, age: Int, salary: Int): Observable() {
    var age by ObservableProperty(age, this)
    var salary by ObservableProperty(salary, this)
}
```
- by 오른쪽의 객체를 위임 객체라고 부르며
- 실제로 해당 프로퍼티에 접근할때 위임 객체의 setValue와 getValue가 호출된다
- 하지만 이렇게 직접 구현하지 않아도 코틀린 표준 라이브러리에서 Delegates Singleton 객체를 제공한다

- https://cs.android.com/android-studio/kotlin/+/master:libraries/stdlib/src/kotlin/properties/Delegates.kt;l=13?q=Delegates&sq=
- Delegates.notNull => notnull로 제한, lateinit과 유사하지만 성능이 좋지 않아 잘 사용하지 않음
- Delegates.observable => 값이 변경될때마다 수행할 로직 작성 가능
- Delegates.vetoable => observable과 유사하지만 리턴값이 Unit인 observable과 달리 람다의 리턴값이 Boolean이며 해당 로직이 true일때만 값이 set된다
- Delegates.observable 사용 예시
```.kt
class Person(...) {
    var age by Delegates.observable(age) { property: KProperty<*>, oldValue: Any?, newValue: Any? ->
        //값이 변경될때마다 수행할 원하는 로직 작성
    }
    var age by Delegates.vetoable(age) { property: KProperty<*>, oldValue: Any?, newValue: Any? ->
        //값이 변경될때마다 수행할 원하는 로직 작성
        age % 2 == 0 // age가 짝수일때만 setValue가 수행됨, 이외에는 무시
    }
    var age by Delegates.notNull<Int> // 나중에 값 초기화 가능, null 불가능
}
```
***참고 ReadWriteProperty***
- operator fun setValue, getValue를 직접 정의해도 되지만 컴파일러 또는 IDE가 이를 자동으로 완성해주지 않아 불편한데, ReadWriteProperty라는 인터페이스를 상속받는 방법이 있다
- ReadWriteProperty에 operator fun setValue, getValue가 정의되어 있다
- ProvideDelegateProvider를 상속받아 provideDelegate를 override 할 수 있다
- https://cs.android.com/android-studio/kotlin/+/master:libraries/stdlib/src/kotlin/properties/Interfaces.kt;l=38?q=ReadWriteProperty
- 실제로 사용하면 아래처럼 override를 통해 구현할 수 있다

```.kt
class Holder<T: Any>: ReadWriteProperty<Any?, T?>, PropertyDelegateProvider<Any?, T?> {
    override fun getValue(thisRef: Any?, property: KProperty<*>): T? {
        TODO("Not yet implemented")
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: T?) {
        TODO("Not yet implemented")
    }

    override fun provideDelegate(thisRef: Any?, property: KProperty<*>): T? {
        TODO("Not yet implemented")
    }
}
```
