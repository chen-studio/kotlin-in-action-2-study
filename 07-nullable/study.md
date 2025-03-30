## 0. Objectives
- 코틀린의 nullable 타입에 대한 이해

## 1-1 NullPointerException을 피하고 값이 없는 경우 처리: nullable
- 코틀린은 NullPointerException을 피하기 위한 nullable 타입이 별도로 존재
- 코틀린은 이 방법으로 null에 대한 접근 방법을 컴파일 시점으로 옮겨 컴파일러가 널처리를 할 수 있도록 함

## 1-2 널이 될 수 있는 타입으로 널이 될 수 있는 변수 명시
```.java
int strLen(String s) {
  return s.length();
}
```
- 이 자바 함수의 파라미터로 null을 전달하면 NPE가 발생한다
- 이 함수를 코틀린으로 다시 쓸때는 파라미터로 null을 받을 수 있는지 고려해야 한다
```.kt
fun strLen(s: String) = s.length // null을 파라미터로 받지 못하는 경우

fun main() {
  strLen(null) // 컴파일 에러 
}
```
- 이 함수가 null을 파라미터로 받을 수 있는 경우 타입 뒤에 물음표를 명시 해야 한다
```.kt
fun strLenSafe(s: String?) = ...
```
- String? Any? MyCustomType등 어떤 타입이든 타입 이름 뒤에 물음표를 붙이면 그 타입의 변수나 프로퍼티에 null 참조를 저장할 수 있다
- 어떤 널이 될 수 있는 타입의 값이 있다면 그 값에 대해 수행할 수 있는 연산의 종류가 제한된다.
```.kt
fun strLenSafe(s: String?) = s.length()
// ERROR: only safe (?.) or non-null asserted(!!.) calls are allowed
```
- 널이 될 수 있는 값을 널이 될 수 없는 타입의 변수에 대입할 수 없다
```.kt
val x: String? = null
var y: String = x // ERROR: Type mismatch
```
- 이렇게 여러 제약을 피하고 코드를 작성하는 방법은 현재 변수가 null인지 비교하는 것이다
```.kt
fun strLenSafe(s: String?): Int = if(s != null) s.length else 0
```
- 널이 될수 있는 값을 비교하는 방법이 if뿐이라면 코드가 매우 복잡해질 것이다. 아래에서 다른 방법을 알아보자

## 1-3 타입의 의미 자세히 살펴보기
- (그리 와닿지 않는 내용 중략)
- 자바의 String은 null과 String 두 종류의 값을 가질 수 있다
- 이 두종류의 값은 완전히 다르다 심지어 자바의 instanceof 연산자도 null이 String이 아니라고 한다
- 이는 자바의 타입 시스템이 null을 제대로 다루지 못한다는 것이다
- 자바에도 @NotNull @Nullable의 annotation을 통해서 어느정도 null처리를 다루기도 한다

## 1-4 안전한 호출 연산자로 null검사와 메서드 호출 합치기 :?.
- 안전한 호출 연산자 `?.` 를 사용하면 null검사와 메서드 호출을 한번에 수행한다
```.kt
str?.uppercase()
if(str != null) str.uppercase else null
```
- 안전한 호출 연산자를 사용한 결과 타입도 nullable 타입이라는 사실에 유의하자
- 메서드 호출뿐 아니라 프로퍼티를 읽거나 쓸 때도 안전한 호출을 사용할 수 있다
```.kt
class Employee(val name: String, val manager: Employee?)
fun managerName(employee: Employee): String? = employee.manager?.name
```
- 객체 그래프에서 널이 될 수 있는 중간 객체가 여럿 있다면 한 식 안에서 안전한 호출을 연쇄해서 사용하면 편하게 사용할 수 있다
```.kt
val country = company?.address?.country
```

## 1-5 엘비스 연산자로 null에 대한 기본값 제공: ?:
- null인 경우 null 대신 사용할 기본값을 지정할때 사용할 수 있는 엘비스 연산자를 제공한다
```.kt
fun greet(name: String?) {
  val recipient: String = name ?: "unnamed"
}
```
- 이 연산자는 두가지 값을 받는데, 첫 번째 값이 null이 아니면 그 값이 전체의 결과이고 null이면 두번째 값이 결과이다
- 안전한 호출 연산자와 엘비스 연산자를 함께 사용하면 좀 더 간결한 코드를 작성할 수 있다
```.kt
fun strLenSafe(s: String?): Int = s?.length ?: 0
fun Person.countryName() = company?.address?.country ?: "Unknown"
```
- 코틀린에서는 return이나 throw등도 식이기 때문에 엘비스 연산자의 오른쪽에 return, throw등을 사용할 수 있어 편리하다
- 이런 패턴은 함수의 전제조건을 검사하는 경우 유용하게 사용할 수 있다
```.kt
class Address(...)

class Company(val name: String, val address: Address?)

class Person(val name: String, val companyt: Company?)

fun printShippingLabel(person: Person) {
val address = person.company?.address ?: throw IllegalArgumentException("No address") // 주소가 없으면 예외 발생
with(address) { // address는 이제 null이 아니다
  ...
}
```

## 1-6 예외를 발생시키지 않고 안전하게 타입을 캐스트하기: as?
- 코틀린의 타입 캐스트 연산자인 as와 비슷하게 nullable 캐스트가 가능한 as?가 있다
- 물론 as를 사용할때마다 is를 통해 as로 변환 가능한 타입인지 검사하는 방법도 있지만 as? 라는 좀 더 나은 솔루션이 존재한다
- other as? Person => 캐스팅 가능한 경우 other as Person 수행, other !is Person인 경우 null
```.kt
class Person(val firstName: String, val lastName: String) {
  override fun equals(o: Any?): Boolean {
    val otherPerson = o as? Person ?: return false

    return otherPerson.firstName == firstName && otherPerson.lastName == lastName
  }
}
```

## 1-7 널 아님 단언: !!
- !!은 코틀린에서 널이 될 수 있는 타입의 값을 다룰 때 사용할 수 있는 도구 중에서 가장 단순하면서도 무딘 도구다
- 느낌표를 이중으로 사용하면 어떤 값이든 널이 아닌 타입으로 바꿀 수 있다
```.kt
fun ignoreNulls(str: String?) {
  val strNotNull: String = str!! // 널인 경우 NullPointerException 발생
}
```
- !!는 컴파일러에게 나는 이 값이 null이 아님을 잘 알고 있다. 예외가 발생해도 감수하겠다 라고 말하는 것이다
- !!에서 발생한 예외는 어떤 파일의 몇 번째 줄인지에 대한 정보는 들어있지만 어떤 식에서 예외가 발생했는지에 대한 정보는 들어있지 않다
- 따라서 아래와 같이 사용하는 것을 피해야 한다
```.kt
person.company!!.address!!.country <- 이런식의 코드 작성은 피하자
```
- (개인적의견) 실제로 !! 보다는 별도의 에러 처리나 디버깅에 유용한 requireNotNull() 또는 checkNotNull()을 더 많이 사용한다
- https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/require-not-null.html
- https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/check-not-null.html

## 1-8 let
- let을 사용하면 널이 될 수 있는 식을 더 쉽게 다룰 수 있다
- 예를들어 널이 될 수 있는 값을 널이 아닌 값만 인자로 받는 함수에 넘기고 싶을때 주로 사용한다
```.kt
fun sendEmailTo(email: String)

fun main() {
  val email: String? = "foo@bar.com"
  sendEmailTo(email) // ERROR: Type mismatch

  email?.let { email -> sendEmailTo(email) }
  email?.let { sendEmailTo(it) }
  email?.let(::sendEmailTo)
}
```

## 1-9 직접 초기화하지 않는 널이 아닌 타입: 지연 초기화 프로퍼티
- 객체를 일단 생성한 다음 나중에 전용 메서드를 통해 초기화하는 프레임워크가 많이 있다
- 이 경우 매번 nullable타입으로 선언하고 초기값을 null로 설정하는 경우가 있다.
- 하지만 이 값을 사용할때마다 null체크를 해야한다는 불편함이 있다
```.kt
class MyTest {
  private var myService: MyService? = null

  @BeforeAll
  fun setUp() {
    myService = MyService()
  }

  @Test
  fun testAction() {
    assertEquals(... myService!!.performAction()) // ?. 또는 !!로 널 체크를 해야함
  }
```
- 이런 문제를 해결하기 위해 객체를 지연 초기화 하는 방법이 있다
- lateinit var를 사용하면 객체를 나중에 초기화할 수 있다
```.kt
class MyTest {
  private lateinit var myService: MyService

  @BeforeAll
  fun setUp() {
    myService = MyService()
  }

  @Test
  fun testAction() {
    assertEquals(... myService.performAction()) // ?. 또는 !!로 널 체크 없이 사용가능
  }
```
- 다만 프로퍼티 초기화 이전에 해당 변수에 접근하면 `UninitializedPropertyAccessException`이 발생한다
- (추가) 해당 변수가 초기화되었는지를 확인할 수 있는 표준 함수도 존재한다
```.kt
if(::myService.isInitialized) {
  ...
}
```

## 1-10 안전한 호출 연산자 없이 타입 확장: 널이 될 수 있는 타입에 대한 확장
- 널이 될 수 있는 타입에 대한 확장 함수를 정의하면 null값을 다루는 강력한 도구로 활용될 수 있다
- 실제로 String? 타입의 수신 객체에 대해 호출할 수 있는 isEmptyOrNull이나 isBlankOrNull 메서드가 있다
```.kt
fun verifyUserInput(input: String?) {
  if(input.isNullOrBlank()) {
    ...
  }
}
```
- 안전한 호출(?.) 없이도 널이 될 수 있는 수신 객체 타입에 대해 선언된 확장 함수를 호출 가능하다
```.kt
input.isNullOrBlank() // 안전한 호출 없음

fun String?.isNullOrBlank(): Boolean = this == null || this.isBlank()
```

## 1-11 타입 파라미터의 널 가능성
- 코틀린에서 함수나 클래스의 모든 타입 파라미터는 기본적으로 null이 될 수 있다
```.kt
fun <T> printHashCode(t: T) {
  println(t?.hashCode()) // t가 널이 될 수 있으므로 안전한 호출을 써야함
}

fun main() {
  printHashCode(null) // T의 타입은 Any?로 추론됨
}
```
- 타입 파라미터에 null을 허용하고 싶지 않다면 upper bound를 지정해야 한다
```.kt
fun <T: Any> printHashCode(t: T) {
  println(t.hashCode())
}
fun main() {
  printHashCode(null) // ERROR
}
```

## 1-12 널 가능성과 자바
- 자바와 코틀린은 상호 운용 가능한 언어지만 자바는 nullable을 지원하지 않는다.
- 이런것들이 가능한 이유는 자바는 @Nullable, @NotNull등의 어노테이션을 사용할 수 있다
- @Nullable + Type = Type?
- @NotNull + Type = Type
- 코틀린은 javax.annotation, android.support.annotation, org.jetbrains.annotations등의 어노테이션으로 자바의 nullable타입을 제어한다
- 이런 어노테이션이 없는 경우는 `플랫폼타입` 이 된다

## 1-13 플랫폼타입
- 코틀린에서 자바를 사용할때 타입이 null인지 아닌지 알 수 없는 경우, 느낌표가 붙은 타입인 플랫폼 타입이 된다(Type!)
- 어떤 플랫폼 타입의 값이 널이 될 수도 있음을 알고 있다면 그 값을 사용하기 전에 null인지 검사할 수 있다
- 만약 널이 아님을 알고 있다면 null검사없이 직접 사용해도 되지만 null인 경우 NPE가 발생할 수 있다
```.java
public class Person {
  private final String name;

  public Person(String name) {
    this.name = name;
  }

  public String getName() {
    return name;
  }
}
```

```.kt
fun yellAt(person: Person) {
  println(person.name.uppercase() + "!!!")
}

fun main() {
  yellAt(Person(null)) // NPE 발생
}
```

```.kt
fun yellAt(person: Person) {
  println((person.name ?: "AnyOne").uppercase() + "!!!") // 정상 동작
}

fun main() {
  yellAt(Person(null)) // NPE 발생 X,
  // ANYONE!!!
}
```
- 코틀린이 이처럼 플랫폼 타입을 도입한 이유는
- 모든 자바 타입을 nullable로 다룰 수 있지만 매번 null 검사를 처리해야 한다는 불편함이 있다
- 이 문제는 제네릭에서 더 드러나는데 자바의 ArrayList<String>을 코틀린에서 ArrayList<String?>? 처럼 다루면 사용이 매우 까다로워진다
- 따라서 이 값이 null인지 아닌지 알수 없음을 알려주는 플랫폼 타입을 도입하되 프로그래머에게 그 책임을 부여하는 실용적인 접근방법을 선택했다
- 플랫폼 타입은 직접 선언할 순 없지만 컴파일러 오류 메시지에서 종종 볼 수 있다
```.kt
val i: Int = person.name
// ERROR: Type mismatch: inffered type is String! but Int was expected
```
- 플랫폼 타입은 nullable로 다룰 수도 있고 notnull로도 다룰 수 있다
```.kt
val s: String? = person.name
val s1: String = person.name
```

## 1-14 상속
- 코틀린에서 자바 메서드를 오버라이딩 할때 그 메서드의 파라미터와 반환 타입을 nullable로 선언할지 not null로 선언할지 결정해야 한다
```.java
interface StringProcessor {
  void process(String value);
}
```
```.kt
class StringPrinter: StringProcessor {
  override fun process(value: String) {
    println(value)
  }
}

class NullableStringPrinter: StringProcessor {
  override fun process(value: String?) {
    if(value != null) println(value)
  }
}
```
- 상황에 맞게 사용
