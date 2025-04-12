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
- 
