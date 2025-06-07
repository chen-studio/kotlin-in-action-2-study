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

```

