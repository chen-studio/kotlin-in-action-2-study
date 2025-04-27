## 0 Objectives
- 함수 타입
- 고차 함수
- 인라인 함수
- 비로컬 return과 레이블
- 익명함수

## 1 다른 함수를 인자로 받거나 반환하는 함수 정의: 고차 함수
- 고차 함수는 다른 함수를 인자로 받거나 반환하는 함수
- 코틀린에서는 람다나 함수 참조를 사용해 함수를 값으로 표현할 수 있음
- filter도 함수를 인자로 받으므로 고차 함수 이다
```.kt
list.filter { it > 0 }
```

### 1-1 함수 타입은 람다의 파라미터 타입과 반환 타입을 지정한다
- 함수 타입이 어떻게 선언되는지 알아보자
```.kt
val sum = { x: Int, y: Int -> x + y }
val action =  { println(42) }
// 아래는 컴파일러가 추론한 함수의 타입
val sum: (Int, Int) -> Int = { x: Int, y: Int -> x + y } // Int형 파라미터를 2개 받고 Int를 return
val action: () -> Unit = { println(42) } // 파라미터, 리턴값 모두 없는 함수
```
`(Int, String) -> Unit`
--------------   ------
  파라미터 타입      리턴타입
- 람다로 리턴값이 없는 함수를 정의할땐 함수와 다르게 `Unit`을 명시해야 한다
- { x, y -> x + y } 처럼 람다의 파라미터 타입은 생략해도 된다
- 함수 타입에서도 nullable타입을 사용할 수 있다
```.kt
var canReturnNull: (Int, Int) -> Int? = { x, y -> null } 
```
- 널이 될 수 있는 함수 타입 변수를 정의할 수 있다 (변수 자체가 null이거나 함수인 경우)
```.kt
var funOrNull: ((Int, Int) -> Int)? = null
```

### 1-2 인자로 전달 받은 함수 호출
- 인자로 받은 연산을 수행하는 방법을 알아보자
```.kt
fun twoAndThree(operation: (Int, Int) -> Int) {
    val result = operation(2, 3)
    println("The result is $result")
}
fun main() {
    twoAndThree { a, b -> a + b } // 5
    twoAndThree { a, b -> a * b } // 6
}
```
- 인자로 받은 함수를 호출하는 방법은 일반 함수와 동일하다
- 함수 타입에서도 파라미터 이름을 지정할 수 있다
```.kt
fun twoAndThree(operation: (operandA: Int, operandB: Int) -> Int) {
    val result = operation(2, 3)
    println("The result is $result")
}
fun main() {
    twoAndThree { operandA, operandB -> operandA + operandB } // 5
    twoAndThree { alpha, beta -> alpha * beta } // 6
}
```
- 다만 파라미터 이름은 타입 검사 시 무시된다. 따라서 가독성을 위한 기능이라고 생각하면 된다
- filter함수를 통해 더 자세히 알아보자
```.kt
fun String.filter(predicate: (Char) -> Boolean): String
```
- Char를 파라미터로 받아 Boolean값을 return하는 predicate라는 함수 타입을 받고 있다
- 문자가 사라지기를 원하면 false, 유지하기를 원하면 true를 리턴하도록 구현하면 된다

### 1-3 자바에서 코틀린 함수 타입 사용
- 함수 타입을 사용하는 코틀린 코드를 자바에서 어떻게 사용할 수 있는지 알아본다
- 먼저 자바의 람다는 자동으로 코틀린 함수 타입으로 변환된다
```
/* Kotlin */
fun processTheAnswer(f: (Int) -> Int) {
    println(f(42))
}
/* Java */
processTheAnswer(number -> number + 1);
```
- 단 자바에서 코틀린의 함수 타입을 사용할때는 수신 객체(number)를 명시적으로 전달해야 한다
- Unit을 반환하는 함수나 람다를 자바로 작성할 수 있다
- 하지만 코틀린의 Unit타입을 명시적으로 반환해야 한다
```.java
public static void main(String[] args) {
    List<String> strings = new ArrayList();
    strings.add("42");
    CollectionKt.forEach(strings, s -> {
        System.out.println(s);
        return Unit.INSTANCE;
    })
}
```
- 코틀린의 함수 타입은 일반 인터페이스이다. 함수 타입의 변수는 FunctionN 인터페이스를 구현하도록 되어 있다
- 함수 파라미터의 개수(N)에 따라 FunctionN 형태의 인터페이스를 사용하며 인자를 받지 않는 Function0<R>도 있다
```.kt
interface Function1<P1, out R> {
    operator fun invoke(p1: P1): R
}

fun processTheAnswer(f: Function1<Int, Int>) {
    println(f.invoke(42))
}
```
- 이처럼 함수 타입이 단순히 코틀린의 인터페이스이기 때문에 인터페이스를 사용할 수 있는 곳에 함수 타입을 사용해도 된다.
- 예를들어 클래스는 FunctionN 인터페이스나 그와 동등한 함수 타입을 상속할 수 있다(실제로 거의 사용하지 않음)
```.kt
class Adder: (Int, Int) -> Int {
    override operator fun invoke(
        p1: Int,
        p2: Int
    ): Int {
        return p1 + p2
    }
}
```
- FunctionN 인터페이스는 컴파일러가 생성한 합성 타입이며 코틀린 표준 라이브러리에는 존재하지 않는다
- 대신 컴파일러가 필요할 때 이런 인터페이스들을 생성해 준다

### 1-4 함수 타입의 파라미터에 대해 기본값을 지정할 수 있고 널이 될 수도 있다
- 람다를 함수의 파라미터로 받는 경우, 람다에 대한 기본값을 지정할 수 있다
```.kt
fun <T> collection<T>.joinToString(
    separator: String = ", ",
    prefix: String = "",
    postfix: String = "",
    tansform: (T) -> String = { it.toString() }
): String { ... }
```

- 널이 될 수 있는 람다를 함수 파라미터로 받는 것도 가능하다
```.kt
fun foo(callback: (() -> Unit)?) {
    if (callback != null) {
        callback()
    }
    // or 
    callback?.invoke()
}
```

### 1-5 함수를 함수에서 반환 
- 함수가 함수를 반환하는 경우는 실제로 요구사항이 적지만 사용할 수 있다
```.kt
fun getPredicate(): (Person) -> Boolean {
    val startsWithPrefix = { p: Person -> 
        p.firstName.startsWith(prefix) || p.lastName.startsWith(prefix)
    }

    if(!onlyWithPhoneNumber) {
        return statsWithPrefix
    }
    return { startsWithPrefix(it)
        && it.phoneNumber != null
    }
}
```

### 1-6 람다를 활용해 중복을 줄여 코드 재사용성 높이기
- 웹 사이트 방문 기록을 분석하는 예이다
- SiteVisit에는 방문한 사이트의 경로, 머문 시간, 사용자의 운영체제가 들어있다
```.kt
data class SiteVisit(
    val path: String,
    val duration: Double,
    val os: OS
)
enum class OS { WINDOWS, LINUX, MAC, IOS, ANDROID }
val log = listOf(
    SiteVisit("/", 34.0, OS.WINDOWS),
    SiteVisit("/", 22.0, OS.MAC),
    SiteVisit("/login", 12.0, OS.WINDOWS),
    SiteVisit("/signup", 8.0, OS.IOS),
    SiteVisit("/", 16.3, OS.ANDROID),
)
```
- 윈도우 사용자의 평균 방문시간을 출력해보자
```.kt
val averageWindowsDuration = log
    .filter { it.os == OS.WINDOWS }
    .map(SiteVisit::duration)
    .average()
```
- 맥 사용자에 대해 같은 통계를 구하고 싶을때 중복을 피하기 위해 OS를 파라미터로 뽑아낸다
```.kt
fun List<SiteVisit>.averageDurationFor(os: OS) = 
    filter { it.os == os }.map(SiteVisit::duration).average()
```
- 하지만 만약 모바일 디바이스(IOS + Android) 의 평균만을 구하고 싶다면 위 함수는 적절하지 않다
- 따라서 조건을 람다로 받는다
```.kt
fun List<SiteVisit>.averageDurationFor(predicate: (SiteVisit) -> Boolean) = 
    filter(predicate).map(SiteVisit::duration).average()
```
