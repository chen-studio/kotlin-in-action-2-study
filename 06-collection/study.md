## 0.Objectives 
- 컬렉션과 시퀀스에 대한 이해

## 1. 컬렉션에 대한 함수형 API
- 코틀린 표준 라이브러리 함수 중 컬렉션을 다루다가 사용하게 될 함수들에 대해 알아본다

### 1-1. 원소 제거와 변환: filter와 map
- filter와 map 함수는 컬렉션을 다루는 토대가 된다
- 각 함수에 대해서는 숫자를 사용하는 예제와 아래 Person 클래스를 사용하는 예제를 사용한다
```.kt
data class Person(val name: String, val age: Int)
```
- filter 함수는 컬렉션을 순회하면서 주어진 람다가 true를 반환하는 원소들만 필터링한다.
- 숫자 필터링
```.kt
val list = listOf(1, 2, 3, 4)
println(list.filter { it % 2 == 0 }) // 짝수만 필터링, 2와 4
```
- 나이가 30살 이상인 사람 필터링
```.kt
val people = listOf(Person("A", 20), Person("B", 33))
println(people.filter { it.age >= 30 }) // 33살인 B만 남는다
```
- filter 함수는 주어진 조건과 일치하는 원소들로 이루어진 새 컬렉션을 만들지만 그 과정에서 원소를 변환하진 않는다
- 예제의 결과로 나온 값도 여전히  Person 객체다.
- 이와달리 map 함수는 컬렉션의 원소를 변환할 수 있게 해준다
- 또한 map 함수의 결과를 같은 개수의 원소가 들어있는 새 컬렉션이다. 길이는 같지만 각 원소는 주어진 함수에 따라 변환된다
```.kt
val list = listOf(1, 2, 3, 4)
println(list.map { it * it }) // 1, 4, 9, 16
```
- Person에서 사람의 리스트가 아닌 이름의 리스트를 출력하고 싶다면 map으로 이름만 추출된 새 컬렉션을 얻을 수 있다
```.kt
people.map { it.name }
people.map(Person::name) // 람다에서 배운 멤버참조 사용
```
- 이런 호출을 연결하여 30살 이상인 사람의 나이를 출력하면 아래처럼 사용할 수 있다
```.kt
people.filter { it.age > 30 }.map(Person::name)
```
- 이제 이 리스트에서
