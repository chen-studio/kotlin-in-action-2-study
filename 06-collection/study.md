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
- 이런 호출을 연결하여 30살 이상인 사람의 나이를 출력하면 아래처럼 작성할 수 있다
```.kt
people.filter { it.age > 30 }.map(Person::name)
```
- 리스트에서 가장 나이 많은 사람의 이름을 알고 싶다고 하면 아래처럼 작성할 수 있다
- 먼저 리스트에서 나이가 가장 많은 사람을 한명 찾고 그 사람과 나이가 같은 모든 사람을 찾는다
```.kt
people.filter {
  val oldestPerson = people.maxByOrNull(Person::age)
  it.age == oldestPerson?.age
}
```
- 하지만 이 코드는 리스트에서 나이가 가장 많은 사람을 찾는 작업을 반복해서 수행한다는 단점이 있다
- 만약 100명의 사람이 있다면 최대 age를 찾는 작업을 100번 수행한다
- 이를 좀더 개선해 최댓값을 한번만 찾게 해보자
```.kt
val maxAge = people.maxByOrNull(Person::age)?.age
people.filter { it.age == maxAge }
```
- 꼭 필요하지 않은 경우 굳이 계산을 반복하지 말자
- 이렇게 컬렉션을 사용할때 람다를 사용하면 코드 자체는 더 간결해 질 수 있지만 내부에서 불합리한 계산식이 일어날 수 있으므로 주의하자
- 만약 filter, map 함수를 수행할때 각 항목의 index가 필요하다면 filterIndexed또는 mapIndexed를 사용하면 된다
```.kt
numbers.filterIndexed { index, element -> index % 2 == 0 && element > 3 }
numbers.mapIndexed { index, element -> index + element }
```
- 맵을 사용할땐 key와 value를 필터링하는 별도의 함수가 존재한다.
- filterKeys, mapKeys, filterValues, mapValues
```
val numbers = mapOf(0 to "zero", 1 to "one")
numbers.mapValues { it.value.uppercase() }
```
### 1-2. 컬렉션 값 누적: reduce와 fold
- 이런 함수를 사용할땐 accumulator라는 개념이 등장한다
- 이 함수들은 컬렉션을 받아서 하나의 값을 반환한다
- reduce는 예제로 살펴보자
```.kt
val list = listOf(1, 2, 3, 4)
list.reduce { acc, element -> acc + element }
// 실제 동작
// STEP1: list의 첫번째 원소 1이 accumulator라고 하는 공간에 저장된다
// STEP2: list의 두번째 원소 2에 대한 reduce 동작을 수행할때, reduce의 첫번째 param인 acc로 accumulator에 있던 1이 전달된다.
// 따라서 블록의 결과값은 acc(1) + 현재원소(2) = 3이 되고, 1대신 3이 accumulator에 저장된다
// STEP3: list의 세번째 원소 3과 acc(3)을 더한 6이 acc에 저장된다
// STEP4: list의 네번째 원소 4와 acc(6)을 더한 10이 최종 결과로 반환된다
```
- fold함수는 이와 비슷하지만 STEP1에서 첫번째 원소에 대해 동작을 수행하기 위한 임의의 시작값을 지정할 수 있다
```.kt
val list = listOf(1, 2, 3, 4)
list.fold(10) { acc, element -> acc + element }
// 실제 동작
// STEP1: list의 첫번째 원소 1과 초기 acc 값으로 지정한 10을 더해 11이 acc에 저장된다
// STEP2: list의 두번째 원소 2에 acc(11)을 더한 13이 acc에 저장된다
// STEP3: list의 세번째 원소 3과 acc(13)을 더한 16이 acc에 저장된다
// STEP4: list의 네번째 원소 4와 acc(16)을 더한 20이 최종 결과로 반환된다
```
- reduce와 fold의 STEP별 결과를 모두 알고 싶다면 runningReduce, runningFold를 사용할 수 있다
- 이들의 유일한 차이점은 최종 결과값만 반환하는 것이 아닌 중간단계를 포함한 결과를 list로 반환한다는 점이다
```.kt
val list = listOf(1, 2, 3, 4)
list.runningReduce { acc, element -> acc + element } // [1, 3, 6, 10] 리스트 반환
list.runningFold(10) { acc, element -> acc + element } // [11, 13, 16, 20] 리스트 반환
```

### 1-3. 컬렉션에 술어 적용: all, any, none, count, find
- 컬렉션에 대해 자주 수행하는 연산으로, 컬렉션의 모든 원소가 어떤 조건을 만족하는지 판단하는 연산이다
- 예를들어, 그룹의 모든 사람의 나이가 27살 이하인지 판단하려면 아래와 같이 작성할 수 있다
```.kt
val people = listOf(Person("A", 20), Person("B", 33))
people.all { it.age <= 27 } // false
```
- 만족하는 원소가 하나라도 있는지 궁금할때는 any를 사용한다
```.kt
people.any { it.age <= 27 } // true
```
- 만족하는 원소가 하나도 없을때 true를 반환하는 none도 있다
```.kt
people.none { it.age >= 44 } // true
```
- 이런 함수들이 빈 컬렉션에 대해서 어떻게 동작할까 궁금할 수 있다
```.kt
emptyList<Int>().any { it > 42 } 만족하는 원소가 없으므로 false
emptyList<Int>().none { it > 42 } 만족하는 원소가 없으므로 true
emptyList<Int>().all { it > 42 } all은 빈컬렉션에 대해 항상 true를 반환한다
// all은 컬렉션에 조건에 일치하지 않는 원소가 하나라도 있는지 본다. 빈 컬렉션의 경우 조건에 일치하지 않는 원소가 없어 true를 반환한다

public inline fun <T> Iterable<T>.all(predicate: (T) -> Boolean): Boolean {
    if (this is Collection && isEmpty()) return true
    for (element in this) if (!predicate(element)) return false
    return true
}
```
- 조건을 만족하는 원소의 개수를 알고 싶다면 count를 사용한다
```.kt
people.count { it.age <= 27 } // 1

people.filter { it.age <= 27 }.size // 이 동작은 filter의 결과로 원소들을 저장하지만, count는 단순 개수만 세기 때문에 훨씬 효율적이다
```
- 조건을 만족하는 원소를 하나 찾고 싶을땐 find를 사용한다
```.kt
people.find { it.age <= 27 } // Person("A", 20)
```
- find는 조건을 만족하는 원소가 있는 경우 첫번째 원소를 반환하며 없는경우 null을 반환한다
- find와 firstOrNull은 동일하다

### 1.4 리스트를 분할해 리스트의 쌍으로 만들기: partition
- 컬렉션을 어떤 조건을 만족하는 그룹과 그렇지 않은 그룹으로 나누어야 한다면 filter와 filterNot을 사용해서 두번 연산을 진행할 것이다
- 하지만 이를 더 간결하게 처리하는 방법이 partition 함수다
```.kt
val (groupA, groupB) = people.partition { it.age <= 27 } // 두개의 그룹으로 분리

public inline fun <T> Iterable<T>.partition(predicate: (T) -> Boolean): Pair<List<T>, List<T>> {
    val first = ArrayList<T>()
    val second = ArrayList<T>()
    for (element in this) {
        if (predicate(element)) {
            first.add(element)
        } else {
            second.add(element)
        }
    }
    return Pair(first, second)
}
// partition은 Pair로 두개의 List를 반환한다.
// 함수의 결과값이 Pair이고 이 값을 두개의 변수에 할당하고 싶다면, 괄호를 통해 한번에 사용할 수 있다
val (a, b) = pairFunction()
```

### 1.5 리스트를 여러 그룹으로 이루어진 맵으로 바꾸기: groupBy
- partition처럼 컬렉션의 원소들을 참과 거짓으로만 분리할 수 없는경우, groupBy를 사용하면 Map으로 묶을 수 있다
```.kt
val people = listOf(Person("A", 31), Person("B", 29), Person("C", 31))
people.groupBy { it.age } // 결과는 Map<Int, List<Person>> 타입으로 리턴

val list = listOf("apple", "apricot", "banana", "cantaloupe")
list.groupBy(String::first) // String의 확장함수지만 멤버 참조 가능, Map<Char, List<String>>
```

### 1.6 컬렉션을 맵으로 변환: associate, associateWith, associateBy
- groupBy함수는 원소들을 그룹화 하여 value가 List<T> 형태로 저장된다
- 만약 원소를 그룹화 하지 않으면서 컬렉션을 Map으로 변환하고 싶은 경우, associate 함수를 사용한다
```.kt
val people = listOf(Person("A", 31), Person("B", 29), Person("C", 31))
people.associate { it.name to it.age } => Map<String, Int> 로 리턴
```
- person은 그대로 두고 key 또는 value로 특정 요소를 지정하고 싶은 경우 associateWith, associateBy를 사용한다
```.kt
people.associateWith { it.age } // key = person, value = age가 되며 Map<Person, Int>
people.associateBy { it.age } // key = age, value = person이 되며 Map<Int, Person>
```
- 다만 맵에서 key는 유일해야 한다. 이 함수들도 같은 키값을 여러개 사용한다면 마지막 결과가 덮어쓰여진다

### 1.7 가변 컬렉션의 원소 변경: replaceAll, fill
- 일반적으로 함수형 프로그래밍은 불변 컬렉션을 사용하는것을 권장하지만 가변 컬렉션을 사용해야하는 경우가 있다
- 이런 경우에 대비해 가변 컬렉션에서 사용할 수 있는 replaceAll와 fill이 있다
- replaceAll은 지정한 람다로 모든 컬렉션의 원소를 변경한다
- fill은 지정한 파라미터로 모든 원소를 변경한다
```.kt
val names = listOf("Martin", "Samuel")
names.replaceAll { it.uppercase() } // 모두 대문자로 변경 [MARTIN, SAMUEL]
names.fill("change") // 모두 "change"로 변경 [change, change]
```

### 1.8 컬렉션의 특별한 경우 처리: ifEmpty
- 컬렉션이 비어있지 않은 경우에만 어떤 처리를 하는게 필요한 경우가 있다
- ifEmpty를 사용하면 아무 원소도 없을때 기본값을 생성하는 람다를 제공할 수 있다
```.kt
val empty = emptyList<String>()
empty.ifEmpty { listOf("no", "values", "here" } // 리스트가 비어있으므로 람다의 결과 return
```
- 컬렉션은 아니지만 문자열에서 ifBlank라는 함수를 사용할 수 있다
- 코틀린 문자열에서 empty는 정말로 아무런 문자를 가지고 있지 않은 빈 문자열을 의미하고
- blank는 String에 공백 문자만 있는 경우에도 blank라고 판단한다
```.kt
val blankName = "    "
blankName.ifBlank { "blank" } // 문자열이 공백만 있으므로 blank
blankName.ifEmpty { "empty" } // 문자열이 완전히 비어있는것은 아니므로 그대로 blankName의 값
```

