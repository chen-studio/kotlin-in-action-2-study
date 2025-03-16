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
































