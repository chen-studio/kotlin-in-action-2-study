## 0.Objectives
- 도메인 특화 언어 DSL 만들기
- 수신 객체 지정 람다 사용
- invoke 관례 사용
- 기존 코틀린 DSL 예제

## 1. API에서 DSL로: 표현력이 좋은 커스텀 코드 구조 만들기
- DSL의 목표는 코드의 가독성과 유지 보수성을 좋게 유지하는것
- 깔끔한 API를 위해 코틀린에서는 확장함수, 중위 함수(infix), 람다 it, 연산자 오버로딩 등을 사용한다

| Normal                                                           | Simple                                       | 특성                           |
|------------------------------------------------------------------|----------------------------------------------|--------------------------------|
| 1.to("one")                                                      | 1 to "one"                                   | infix                          |
| set.add(2)                                                       | set += 2                                     | operator overloading           |
| map.get("key")                                                   | map["key"]                                   | get method 관례                |
| file.use({ f -> f.read() } )                                     | file.use { it.read() }                       | 람다를 괄호 밖으로 빼내는 관례 |
| sb.append("yes")  sb.append("no")                                | with (sb) {   append("yes")   append("no") } | 수신 객체 지정 람다            |
| val m = mutableListOf<Int>() m.add(1) m.add(2) return m.toList() | return buildList {   add(1)   add(2) }       | 람다를 받는 빌더 함수          |

- 이러한 깔끔한 API에서 더 나아가 DSL을 사용해보자
- 간단한 예시
```.kt
fun createSimpleTable() = createHTML().
  table {
    tr {
      td { +"cell" }
    }
  }
```

### 1-1 도메인 특화 언어
- 이를 이해하려면 선언적 / 명력적 언어의 특성에 대해 알아야 한다
- 명령적 언어: 어떤 연산을 완수하기 위해 필요한 각 단계를 순서대로 정확히 기술하는 언어로 대부분의 프로그래밍 언어가 해당
- 선언적 언어: 원하는 결과를 기술하기만 하고 그 결과를 달성하기 위해 필요한 세부 실행은 언어를 해석하는 엔진에 맡기는 언어로 SQL, 정규식과 같은 DSL 언어가 해당
- DSL의 단점
- DSL과 호스트 애플리케이션의 통합이 어려우며 DSL 자체 문법 차이로 다른 언어의 프로그램 안에 직접 포함 불가
- 별도 파일 또는 문자열로 저장해서 포함 시킬 수 있지만 IDE 지원, 디버깅, 컴파일 시점 검증 등이 제한됨
- DSL 문법 자체를 학습해야 함
- 이런 단점을 해결하기 위해 코틀린에선 내부 DSL을 지원한다

### 1-2 내부 DSL은 프로그램의 나머지 부분과 매끄럽게 통합된다
- 독립적인 문법 구조를 갖는 외부 DSL과는 달리 내부 DSL은 범용 언어로 작성된 프로그랭의 일부이며 동일한 문법을 사용한다
- 따라서 DSL의 장점을 유지하면서 코틀린의 문법을 사용하는 것이다

```.sql
SELECT Country.name, COUNT(Customer.id)
    FROM Country
INNER JOIN Customer
    ON Country.id = Customer.country_id
GROUP BY Country.name
ORDER BY COUNT(Customer.id) DESC
LIMIT 1
```
```.kt
(Country innerJoin Customer)
    .slice(Country.name, Count(Customer.id))
    .selectAll()
    .groupBy(Country.name)
    .orderBy(Count(Customer.id), order = SortOrder.DESC)
    .limit(1)
```

### 1-3 DSL의 구조
- DSL과 일반 API 사이에 잘 정의된 경계는 없음
- infix나 연산자 오버로딩에 의존하기도 함
- 하지만 DSL은 구조 또는 문법적 특징이 있다
- 전형적인 라이브러리는 함수 호출시 아무런 구조가 없으며 한 호출과 다른 호출 사이에 맥락이 유지되지 않음
- 이런 API를 명령-질의(commend-query) API 라고 부른다

```.kt
// DSL
dependencies {
  testImplementation(...)
  implementation(...)
  ...
}

// 일반 명령-질의
project.dependencies.add("testImplementation", ...)
project.dependencies.add("implementation", ...)
```

### 1-4 내부 DSL로 HTML 만들기

```.html
<table>
  <tr>
    <td>1</td>
    <td>one</td>
  </tr>
  <tr>
    <td>2</td>
    <td>two</td>
  </tr>
</table>
```

```.kt
import kotlinx.html.stream.createHTML
import kotlinx.html.*
 
fun createTable() = createHTML().table {
    val numbers = mapOf(1 to "one", 2 to "two")
    for ((num, string) in numbers) {
        tr {
            td { +"$num" }
            td { +string }
        }
    }
}
```

## 2. 구조화된 API 구축: DSL에서 수신 객체 지정 람다 사용
- 수신 객체 지정 람다를 사용하여 DSL을 활용하는법

### 2-1. 수신 객체 지정 람다와 확장 함수 타입
- buildString에 대한 예시

```.kt
fun buildString(
    builderAction: (StringBuilder) -> Unit,
): String {
    val sb = StringBuilder()
    builderAction(sb)
    return sb.toString()
}

fun main() {
    val s = buildString {
        it.append("Hello, ")
        it.append("World!")
    }
    println(s)
    // Hello, World!
}
```
- 파라미터 (it) 대신 수신 객체 전달로 더 간결한 코드 작성
```.kt
fun buildString(
    builderAction: StringBuilder.() -> Unit,
): String {
    val sb = StringBuilder()
    sb.builderAction()
    return sb.toString()
}

fun main() {
    val s = buildString {
        this.append("Hello, ") // this는 모호성을 해결할때만 사용
        append("World!")
    }
    println(s)
    // Hello, World!
}

```
- 아래처럼 확장 함수 타입의 변수를 정의할 수도 있다 (사용할일은 잘 없을듯)
```.kt
val appendExcl: StringBuilder.() -> Unit =
    { this.append("!") }

fun main() {
    val stringBuilder = StringBuilder("Hi")
    stringBuilder.appendExcl()
    println(stringBuilder)
    // Hi!
    println(buildString(appendExcl))
    // !
}
```
- buildString을 apply와 함께 사용하면 단 한줄로 작성할 수 있다
```.kt
fun buildString(builderAction: StringBuilder.() -> Unit): String =
  StringBuilder().apply(builderAction).toString()
```
- apply, with는 아래와 같이 구현되어 있으며 결과를 받아서 쓸 필요가 없다면 두 함수를 서로 바꿔서 쓸 수 있다
```.kt
inline fun <T> T.apply(block: T.() -> Unit): T {
  block()
  return this
}

inline fun <T, R> with(receiver: T, block: T.() -> R): R = receiver.block()
```

### 2-2. 수신 객체 지정 람다를 HTML 빌더 안에서 사용
```.kt
fun createSimpleTable() = createHTML().
  table {
    tr {
      td { +"cell" }
    }
  }
```
- 이 코드는 일반 코틀린 코드이며 어떤 특별한 템플릿 언어 같은 것이 아님
- 이 안에서 수신 객체 지정 람다가 어떻게 사용되는지 보자
```.kt
fun createSimpleTable() = createHTML().
  table { //this: TABLE
    tr { //this: TR
      td { //this: TD
        +"cell"
      }
    }
  }
```
- 실제로는 IDE에서 수신 타입을 시각화할 수 있도록 도와준다
```.kt
open class Tag

class TABLE : Tag {
    fun tr(init: TR.() -> Unit)
}

class TR : Tag {
    fun td(init: TD.() -> Unit)
}

class TD
```
- TABLE, TR, TD는 모두 HTML 생성 코드에 명시적으로 나타나면 안 되는 유틸리티 클래스다
- 따라서 이름을 모두 대문자로 만들어서 일반 클래스와 구분한다
- 나중에 이 안에 DSL에 도움이 될 수 있는 제약을 추가할 것이다
- 수신 객체를 명시해서 함수를 한번 살펴보자
```.kt
fun createSimpleTable() = createHTML().table {
  this@table.tr {
    (this@tr).td {
      +"cell"
    }
  }
}
```
- 수신 객체 지정 람다가 아닌 일반 람다를 사용하면 HTML 생성 코드 구문이 알아볼 수 없을 정도로 난잡해 질 것이다
- 하지만 만약 깊이가 깊은 구조에서는 어떤 식의 수신 객체가 무엇인지 분명하지 않아서 혼동이 올 수 있다
- 예를들어 img 람다 안에서 바깥쪽 a 태그의 href 프로퍼티를 참조할 수 있다
```.kt
createHTML().body {
  a {
    img {
      href = "https://..."
    }
  }
}
```
- 이를 막기위해 코틀린은 @DslMarker 어노테이션을 제공하여 내포된 람다에서 외부 람다의 수신 객체에 접근하지 못하게 제한할 수 있다
```.kt
@DslMarker
annotation class HtmlTagMarker
```
- @HtmlTagMarker 어노테이션이 붙은 선언에는 암시적 수신 객체에 대한 제약이 추가 된다
- @DslMarker 어노테이션이 붙은 영역 안에서는 암시적 수신 객체가 결코 2개가 될 수 없다
- 우리가 작성하는 태그는 모두 Tag 클래스의 하위 클래스 이므로 @HtmlTagMarker를 아래처럼 추가한다
```.kt
@HtmlTagMarker
open class Tag
```
- 이제 아래 코드는 컴파일 에러가 발생한다
```.kt
createHTML().body {
  a {
    img {
      href = "https://..." // 불가능
    }
  }
}
```

```.kt
@DslMarker
annotation class HtmlTagMarker

@HtmlTagMarker
open class Tag(val name: String) {
    private val children = mutableListOf<Tag>() // 모든 내포 태그 저장

    protected fun <T : Tag> doInit(child: T, init: T.() -> Unit) {
        child.init() // 자식 태그 초기화
        children.add(child) // 자식 태그에 대한 참조 저장
    }

    override fun toString() =
        "<$name>${children.joinToString("")}</$name>"
}

fun table(init: TABLE.() -> Unit) = TABLE().apply(init)

class TABLE : Tag("table") {
    fun tr(init: TR.() -> Unit) = doInit(TR(), init)
}

class TR : Tag("tr") {
    fun td(init: TD.() -> Unit) = doInit(TD(), init)
}

class TD : Tag("td")

fun createTable() =
    table {
        tr {
            td {
            }
        }
    }

fun main() {
    println(createTable())
    // <table><tr><td></td></tr></table>
}
```
- 이처럼 수신 객체 지정 람다는 DSL을 만들때 아주 좋은 도구이다

### 2-3. 코트린 빌더: 추상화와 재사용을 가능하게 해준다
- 아래와 같이 u1, li, h2, p 등의 태그를 사용하는 코드가 있다
```.kt
fun buildBookList() = createHTML().body {
    ul {
        li { a("#1") { +"The Three-Body Problem" } }
        li { a("#2") { +"The Dark Forest" } }
        li { a("#3") { +"Death’s End" } }
    }
 
    h2 { id = "1"; +"The Three-Body Problem" }
    p { +"The first book tackles..." }
 
    h2 { id = "2"; +"The Dark Forest" }
    p { +"The second book starts with..." }
 
    h2 { id = "3"; +"Death’s End" }
    p { +"The third book contains..." }
}
```
- 추상화를 통해 아래처럼 불필요한 코드를 줄일 수 있다
```.kt
fun buildBookList() = createHTML().body {
    listWithToc {
        item("The Three-Body Problem", "The first book tackles...")
        item("The Dark Forest", "The second book starts with...")
        item("Death’s End", "The third book contains...")
    }
}
```
- 그렇다면 실제로 listWithToc 함수를 어떻게 구현할까? 아래처럼 만들 수 있다
```.kt
@HtmlTagMarker
class LISTWITHTOC {
    val entries = mutableListOf<Pair<String, String>>()
    fun item(headline: String, body: String) {
        entries += headline to body
    }
}

fun BODY.listWithToc(block: LISTWITHTOC.() -> Unit) {
    val listWithToc = LISTWITHTOC()
    listWithToc.block()
    ul {
        for ((index, entry) in listWithToc.entries.withIndex()) {
            li { a("#$index") { +entry.first } }
        }
    }
    for ((index, entry) in listWithToc.entries.withIndex()) {
        h2 { id = "$index"; +entry.first }
        p { +entry.second }
    }
}
```

## 3. invoke 관례를 사용해 더 유연하게 블록 내포시키기
- invoke 관례를 사용하면 어떤 커스텀 타입의 객체를 함수처럼 호출할 수 있다

### 3-1. invoke 관례를 사용해 더 유연하게 블록 내포시키기
```.kt
class Greeter(val greeting: String) {
    operator fun invoke(name: String) {
        println("$greeting, $name!")
    }
}

fun main() {
    val bavarianGreeter = Greeter("Servus")
    bavarianGreeter("Dmitry") // Greeter 인스턴스를 함수처럼 호출한다
    // Servus, Dmitry!
}
```
- bavarianGreeter("Dmitry")는 내부적으로 bavarianGreeter.invoke("Dmitry")로 컴파일된다

### 3-2. DSL의 invoke 관례: gradle 의존 관계 선언
- 처음에 살펴보았던 코드이다
```.kt
dependencies {
  testImplementation(...)
  implementation(...)
}
```
- 이 코드처럼 내포된 블록 구조를 허용하면서 평평한 호출 구조도 함께 지원하는 API를 만들고 싶다
```.kt
dependencies.implementation(...)
dependencies {
  implementation(...)
}
```
- 아래는 DependencyHandler의 구현에서 최소한의 부분만 보여준다
```.kt
class DependencyHandler {
    fun implementation(coordinate: String) {
        println("Added dependency on $coordinate")
    }

    operator fun invoke(
        body: DependencyHandler.() -> Unit,
    ) {
        body()
    }
}

fun main() {
    val dependencies = DependencyHandler()
    dependencies.implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.0")
    // Added dependency on org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.0
    dependencies {
        implementation("org.jetbrains.kotlinx:kotlinx-datetime:0.5.0")
    }
    // Added dependency on org.jetbrains.kotlinx:kotlinx-datetime:0.5.0
}
```
- 적은 양의 코드지만 invoke 관례를 이용하면 유연성이 훨씬 커진다

## 4. 실전 코틀린 DSL
- 테스팅, 다양한 날짜 리터럴, 데이터베이스 질의 같은 다양한 주제를 살펴보자

### 4-1. 중위 호출 연쇄시키기
- kotest에서는 아래처럼 중위 호출을 활용한다
```.kt
val s = "kotlin".uppercase()
s should startWith("K")

infix fun <T> T.should(matcher: Matcher<T>) = matcher.test(this)
// should 함수는 Matcher 인스턴스를 요구한다

interface Matcher<T> {
  fun test(value: T)
}
fun startWith(prefix: String): Matcher<String> {
  return object: Matcher<String> {
    override fun test(value: String) {
      if(!value.startsWith(prefix) {
        throw AssertionError("$value does not start with $prefix")
      }
    }
  }
}
```

### 4-2. 원시 타입에 대해 확장 함수 정의하기: 날짜 처리
```.kt
val now = Clock.System.now()
val yesterday = now - 1.days
val later = now + 5.hours

val Int.days: Duration
    get() = this.toDuration(DurationUnit.DAYS)

val Int.hours: Duration
    get() = this.toDuration(DurationUnit.HOURS)

val Int.fortnights: Duration get() =
  (this * 14).toDuration(DurationUnit.DAYS)
```

### 4-3. 멤버 확장 함수: SQL을 위한 내부 DSL
- 
