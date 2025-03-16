## 0. Objectives
- 코틀린 람다에 대한 이해

## 1. 람다식과 멤버 참조

### 1-1 코드 블록을 값으로 다루기

- 람다 또는 람다식이란 다른 함수에 넘길 수 있는 작은 코드 조각 (이라고 되어있지만 함수를 변수로 선언했다고 생각하면 됨) 이다
- 코틀린에서는 함수를 다른 함수의 parameter로 넘겨야 하거나 함수를 return하는 경우를 종종 볼 수 있음
- 이 경우 자바에서는 익명 내부 클래스를 사용했으나 매우 번거로움
- 코틀린에서는 람다를 통해 이를 쉽게 해결할 수 있음
- Android에서 버튼을 클릭했을때 동작을 정의하는 OnClickListener 코드를 살펴보자
```.kt
button.setOnClickListener(object: OnClickListener {
    override fun onClick(v: View) {
        println("I was clicked!")
    }
})
```
- 참고로 setOnClickListener와 OnClickListener는 아래와 같이 구현되어 있다

```.java
public void setOnClickListener(@Nullable OnClickListener l)
```

```.java
public interface OnClickLisnter {
    void onClick(View var1);
}
```
- 람다를 사용하면 위의 코드를 아래처럼 간결하게 나타낼 수 있다
```.kt
button.setOnClickListener { println("I was clicked!") }
```

- 자바로 작성된 interface를 parameter로 받으면서 interface의 메소드가 한개인 경우, 이렇게 람다로 대체할 수 있다.

### 1-2 람다와 컬렉션

- 코드에서 중복을 제거하는 것은 코드를 작성할 때 중요한 요소 중 하나이다
- 람다로 인해 코틀린은 Collection을 다룰 때 강력한 기능을 제공하는 여러 표준 라이브러리를 제공하는 것이 가능해졌다
- 아래와 같이 사람의 이름과 나이를 저장하는 Person 클래스가 있다고 가정하자
```.kt
data class Person(val name: String, val age: String)
```
- List<Person>에서 나이가 가장 많은 사람을 찾기 위해서는 List를 직접 순회하면서 가장 나이가 많은 사용자를 검색할 것이다
- 하지만 코틀린에선 이런 방법이 있다
```.kt
fun main() {
  val people = listOf(Person("a", 25), Person("b", 24))
  val oldestPerson = people.maxByOrNull { it.age }
}
```
- maxByOrNull은 리스트 중 람다에 지정된 로직을 수행했을 때 가장 큰 요소를 반환하며 해당하는 값이 없을 경우 null을 반환한다
- 만약 람다가 함수나 프로퍼티에 위임할 경우, 좀더 간단하게 `멤버 참조` 라는 방법을 사용할 수 있다
```.kt
people.maxByOrNull(Person::age)
```

### 1-3 람다식의 문법
- 람다는 항상 중괄호로 싸여 있으며 파라미터를 지정하고 실제 로직이 담긴 본문을 제공한다.
```.kt
{ x: Int, y: Int -> x + y } // x: Int <- 타입 추론이 가능하며 타입 생략 가능
{ x, y -> x + y }
```
- 또한 람다식은 변수에 저장할 수 있으며 함수처럼 호출이 가능하다
```.kt
fun main() {
    val sum = { x: Int, y: Int -> x + y }
    println(sum(1, 2))
}
```
- 람다식을 직접 호출하는 것도 가능하다
```.kt
fun main() {
    { println(42) }()
}
```
- 하지만 이와 같은 구문은 읽기 어렵고 쓸모도 없다.
- 람다를 만들자마자 바로 호출하는것 보단 run을 활용하자
- run은 인자로 받은 람다를 실행해 주는 라이브러리 함수이다
```.kt
run { println(42) }
```

- 코틀린은 람다를 줄여쓸 수 있는 기능을 제공한다. 만약 코틀린이 제공하는 기능을 사용하지 않는 것을 기준으로 maxByOrNull을 사용하면 아래와 같이 될것이다
```.kt
people.maxByOrNull { p: Person -> p.age } // 줄여 쓴 방법
people.maxByOrNull({ p: Person -> p.age }) // 그대로 쓴 방법
```
- 그대로 쓴 방법은 구분자가 너무 많이 쓰여 가독성이 떨어진다.
- 그리고 컴파일러가 문맥으로부터 추론 가능한 인자 타입을 굳이 적을 필요는 없다
- 마지막으로 인자가 단 하나뿐인 경우 인자에 굳이 이름을 붙이지 않아도 된다.

- 먼저 코틀린에서는 람다가 마지막에 오는 경우 람다를 ()괄호 밖으로 뺄 수 있다.
```.kt
people.maxByOrNull() { p: Person -> p.age } 
```
- 또한 람다가 함수의 유일한 인자이고 괄호 뒤에 람다를 썼다면, 호출 시 빈 괄호는 없애도 된다
```.kt
people.maxByOrNull { p: Person -> p.age }
```
- 모두 같은 형태이지만 마지막 람다의 형태가 가장 읽기 쉽다
- 만약 람다가 둘 이상인 경우 두개 이상의 람다를 괄호 밖으로 뺄 수는 없으므로 괄호 안쪽에 넣는것이 더 좋다
- joinToString의 경우를 한번 살펴보자
```.kt
val people = listOf(Person("a", 25), Person("b", 24))
val names = people.joinToString(
    separator = " ",
    transform = { p: Person -> p.name }
) // 이것보다는 아래와 같은 형태가 더 좋다

val names = people.joinToString(" ") { p: Person -> p.name } 
```
- 그리고 IntelliJ나 Android Studio에서 람다를 줄여쓰지 않으면 대부분 노란색으로 Lint 경고를 노출한다. 이 경우 Alt + Enter를 사용하면 간편하게 코드를 최적으로 줄일 수 있다
- 그리고 람다 내 파라미터가 하나인 경우, p: Person처럼 파라미터를 명시하지  않아도 `it`이라는 키워드로 사용할 수 있다
```.kt
people.maxByOrNull { it.age } 
```

### 1-4 현재 영역에 있는 변수 접근
- 코틀린의 람다를 함수 안에서 정의하면 람다보다 앞에 선언된 로컬 변수까지 람다에서 모두 사용할 수 있다.
```.kt
fun printMessageWithPrefix(messages: List<String, prefix: String) {
    var test = "test"
    messages.forEach {
        println("$prefix $it $test")
    }
} // prefix에 접근 가능, tes에접근 가능
```
- 이처럼 코틀린에서는 람다 안에서 람다 밖 함수에 있는 파이널이 아닌 변수에 접근하고 변경할 수 있다.
- 여기서 prefix나 test를 `람다가 캡처한 변수` 라고 부른다
- 만약 어떤 함수가 자신의 로컬 변수를 캡처한 람다를 반환하거나 다른 변수에 저장한다면 로컬 변수의 생명주기와 함수의 생명주기가 달라질 수 있다.
- 캡처한 변수가 있는 람다를 저장해서 함수가 끝난 뒤에 실행해도 람다의 본문 코드는 여전히 사용할 수 있다.
- 이런 동작이 가능한 이유는 val(final)의 경우, 람다 코드를 변수 값과 함께 저장한다. 정확히는 람다는 자바에서 Function이라는 객체로 변환되어 새로운 객체를 생성하여 반환한 것과 똑같다
- 파이널이 아닌 변수(var)은 변수를 특별한 래퍼 클래스로 감싸서 나중에 변경할거나 읽을 수 있도록 만든다
```.kt
class Ref<T>(var value: T)
```

### 1.5 멤버 참조
- 코틀린에서는 자바8과 마찬가지로 함수를 값으로 바꿀 수 있다. 이때 이중 클론(::)을 사용하며 이를 멤버 참조라고 부른다
```.kt
val getAge = Person::age
```
- `::는 클래스 이름과 참조하려는 멤버(프로퍼티 or 함수) 이름 사이에 위치한다
- Person(클래스)::age(멤버)
- 이를 활용하면 람다식을 더 간편하게 표현할 수 있다
```.kt
people.maxByOrNull(Person::age)
people.maxByOrNull { person: Person -> person.age } // 둘은 같은 로직
```
- 또한 최상위에 선언된 함수나 프로퍼티를 참조할 수도 있다
```.kt
fun salute() = println("Salute!")
fun main {
    run(::salute)
}
```
- 이런 경우 클래스 이름을 생략하고 ::로 참조를 바로 시작한다
- 람다가 인자가 여럿인 다른 함수에게 작업을 위임하는 경우, 멤버 참조를 사용하면 아주 편리하다
```.kt
val action = 
```














































































