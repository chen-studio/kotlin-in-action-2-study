## 0. Objectives
- 어노테이션 적용과 정의
- 리플렉션을 사용해 실행 시점에 객체 내부 관찰하기
- 예제

## 1. 어노테이션 선언과 적용
- 어노테이션을 사용하면 선언에 추가적인 메타데이터를 연관시킬 수 있다

### 1-1. 어노테이션을 적용해 선언에 표지 남기기
- 어노테이션을 이용하면 함수나 여러 클래스에 표지를 남길 수 있다
```.kt
@Test
fun testTrue() {...}
```
- `@Deprecated`는 이 선언이 더이상 사용되지 않을 것임을 명시한다
- `@Deprecated`는 최대 아래 3가지 파라미터를 받는다
- message: 사용 중단 예고 이유 설명
- replaceWith: 어떤 것으로 대체 되는지 설명
- level: 사용 중단에 대한 제한 수준
- level WARNING: 단순 경고
- level ERROR, HIDDEN: 컴파일 X
- HIDDEN: 이전에 컴파일된 코드와 호환성만 유지
```.kt
@Deprecated("Use removeAt(index) instead.", ReplaceWith("removeAt(index)"))
fun remove(index: Int) { ... }
```
- `@Deprecated` 가 명시된 함수를 사용하는 경우 인텔리제이에서 경고 메시지와 퀵 픽스도 제공해준다
- 어노테이셔 인자를 지정하는 문법은 자바와 조금 다르다
- `클래스를 어노테이션 인자로 지정`: @MyAnnotation(MyClass::class)처럼 ::class를 클래스 이름 뒤에 넣어야 한다
- 예를들어 직렬화 라이브러리는 @DeserializeInterface(CompanyImpl::class)처럼 역직렬화 과정에서 쓰이는 구현과 인터페이스를 연결하기 위해 클래스를 받는 어노테이션을 제공할 수 있다
- `다른 어노테이션을 인자로 지정`: 인자로 들어가는 어노테이션의 이름 앞에 @를 넣지 말라
- 예를들어 방금 살펴본 예시의 ReplaceWith앞에 @를 사용하지 않는다
- `배열을 인자로 지정`: @RequestMapping(path = ["/foo", "/bar"])처럼 각괄호를 사용한다
- 대신 배열을 지정하기 위해 arrayOf 함수를 사용할 수도 있다
- 어노테이션 인자는 컴파일 시점에 알 수 있어야 한다
- 따라서 프로퍼티를 어노테이션 인자로 사용하려면 const 변경자를 붙여야한다
```.kt
const val TEST_TIMEOUT = 10L

class MyTest {
  @Test
  @Timeout(TEST_TIMEOUT)
  fun testMethod() {...}
}
```

### 1.2 어노테이션 참조 범위 지정: Annotation Target
- 이 어노테이션을 붙일 요소를 지정하는 방법이 있다
- 사용 지점 타깃은 @기호와 어노테이션 이름 사이에 붙으며 어노테이션 이름과는 콜론으로 구분된다
- 예를들어 get이라는 단어는 @JvmName 어노테이션을 프로퍼티게터에 이용하라는 뜻이다
```.kt
@get:JvmName("obtainCertificate")
@set:JvmName("putCertificate")
var certificate: String  "..."

// performCalculation 으로 자바에서 호출이 가능하도록 함
@JvmName("performCalculation")
fun calculate(): Int {
  return (2 + 2) - 1
}
```

in java
```.java
class Foo {
  public static void main(String[] args) {
    var certManager = new CertificateManager();
    var cert = certManager.obtainCertification();
    certManager.putCertificate("...")
  }
}
```
- 사용 지점 타깃 목록
- property: 프로퍼티 전체(자바에서 선언된 어노테이션에는 사용 불가)
- field: 프로퍼티에 의해 생성되는 필드
- get: 프로퍼티 게터
- set: 프로퍼티 세터
- receiver: 확장 함수나 프로퍼티의 수신 객체 파라미터
- param: 생성자 파라미터
- setparam: 세터 파라미터
- delegate: 위임 프로퍼티의 위임 인스턴스를 담아둔 필드
- file: 파일 안에 선언된 최상위 함수와 프로퍼티를 담아두는 클래스
- 클래스 또는 함수 선언이나 타입만 사용할 수 있는 자바와 달리 코틀린에서는 어노테이션 인자로 클래스 또는 함수 선언이나 타입 외에 임의의 식을 허용한다
- 예를들어 컴파일러의 경고를 무시하도록 하는 @Suppress가 있다

#### 자바 API를 어노테이션으로 제어하기
- 코틀린에선 코틀린을 자바 바이트 코드로 컴파일하는 방법과 코틀린 선언을 자바에 노출하는 방법을 제어하기 위한 어노테이션을 많이 제공한다
- 예들들어 자바의 volatile 키워드 대신 @Volatile 어노테이션을 사용한다
- @JvmName은 자바 필드나 메서드 이름을 변경
- @JvmStatic 객체 선언이나 동반 객체의 메서드에 적용하면 메서드가 자바 정적 메서드로 노출
- @JvmOverloads 디폴트 파라미터가 있는 함수에 대해 컴파일러가 자동으로 오버로딩된 함수 생성
- @JvmField 대상 프로퍼티를 게터나 세터가 없는 공개된 자바 필드로 노출
- @JvmRecord data class에 사용하면 자바 레코드 클래스 선언

### 1-3. 어노테이션을 활용해 JSON 직렬화 제어
- 제이키드에 대한 설명

### 1-4. 어노테이션 선언
- 코틀린
```.kt
annotation class JsonExclude

annotation class JsonName(val name: String)
```
- 자바
```.java
public @interface JsonName {
  String value();
}
```
- 자바 어노테이션에는 value라는 메서드가 있다는 점을 기억하고 코틀린에서는 name이라는 프로퍼티를 기억하라
- 자바에서 value를 제외한 모든 애트리뷰트에는 이름을 명시해야 한다
- 코틀린에서는 일반적인 생성자 호출과 동일하다
- 자바에서 선언한 어노테이션을 코틀린에서 사용할떈 value를 제외한 모든 인자에 대해 이름 붙은 인자 구문을 사용해야 한다

### 1-5. 메타어노테이션: 어노테이셔을 처리하는 방법 제어
- 코틀린의 클래스에도 어노테이션을 붙일 수 있으며 어노테이션 클래스에 붙일 수 있는 어노테이션을 `메타 어노테이션` 이라고 부른다
- 가장 흔한 것은 @Target이다
```.kt
@Target(AnnotationTarget.PROPERTY)
annotation class JsonExclude
```
- 예시로 사용한 제이키드 어노테이션의 경우 프로퍼티 어노테이션만 사용하므로 @Target을 꼭 지정해야 한다. 만약 지정하지 않으면 모든 선언에 적용할 수 있다
- @Target(AnnotationTarget.CLASS, AnnotationTarget.METHOD) 처럼 여러개 지정할 수도 있다
- 메타 어노테이션을 직접 만들어야 한다면 ANNOTATION_CLASS를 타깃으로 지정하자
```.kt
@Target(AnnotationTarget.ANNOTATION_CLASS)
annotation class BindingAnnotation

@BindingAnnotation
annotation class MyBinding
```
- target을 PROPERTY로 지정한 어노테이션을 자바 코드에서는 사용할 수 없다
- 자바에선 그런 어노테이션을 사용하려면 AnnotationTarget.FIELD를 두 번째 타깃으로 추가해야 한다
#### @Retention
- retention은 어노테이션 클래스를 소스 수준에서만 유지할지 .class 파일에 저장할지 런타임에서 리플렉션을 통해 접근할 수 있게 할건지 지정하는 방법이다
- 필요한 scope 까지만 지정하면 좋다

### 1-6. 어노테이션 파라미터로 클래스 사용 
```.kt
interface Company {
  val name: String
}

data class CompanyImpl(override val name: String): Company

data class Person(
  val name: String,
  @DeserializeInterface(CompanyImpl::class) val company: Company
)

annotation class DeserializeInterface(val targetClass: KClass<out Any>)
```

- Company::class 타입은 KClass<CompanyImpl>이며 이 타입은 방금 살펴본 DeserializeInterface의 파라미터 타입인 KClass<out Any>의 하위 타입이다
- out (공변) 을 선언해야 CompanyImpl::class를 인자로 넘길 수 있다

### 1-7 어노테이션 파라미터로 제네릭 클래스 받기
- 너무 영양가가 없어 예시를 쓰진 않았지만.. 스타 프로젝션을 사용하면 됩니다
```.kt
annotation class CustomSerializer(
  val serializerClass: KClass<out ValueSerializer<*>>
)
```

## 2. Reflection
- 런타임에 객체의 프로퍼티와 메서드에 접근할 수 있게 해주는 방법
- 코틀린 리플렉션을 다루려면 코틀린 리플렉션 kotlin.reflect와 kotlin.reflect.full 패키지의 코틀린 리플렉션 API를 사용하면 된다

### 2-1 KClass, KCallable, KFunction, KProperty
- 코틀린 리플렉션 클래스는 KClass 타입으로 표현된다
```.kt
class Person(val name: String, val age: Int)

fun main() {
  val person = Person("Alice", 29)
  val kClass = person:class
  println(kClass.simpleName)
  // Person
  kClass.memberProperties.forEach { println(it.name) }
  // age
  // name  
}
```
in KClass
```.kt
interface KClass<T : Any> {
  val simpleName: String?
  val qualifiedName: String?
  val members: Collection<KCallable<*>>
  val constructors: Collection<KFunction<T>>
  val nestedClasses: Collection<KClass<*>>
  ...
}
```
- simpleName과 qualifiedName이 nullable인 이유는 익명 객체 사용 시 해당 값이 null이기 때문이다
- KCallable은 함수와 프로퍼티를 아우르는 공통 상위 인터페이스이다
- KCallable의 call을 사용하면 함수나 프로퍼티의 게터를 호출할 수 있다
```.kt
interface KCallable<out R> {
  fun call(vararg args: Any?): R
  //...
}

fun foo(x: Int) = println(x)

fun main() {
  val kFunction = ::foo
  kFunction.call(42) // 42
}
```
- KCallabale은 vararg 리스트를 파라미터로 받고 실제 그 함수의 파라미터를 정확히 파라미터로 전달해야 한다
- 예를들어 파라미터 1개를 받는 KFunction의 call 함수에 2개의 파라미터를 넣어 호출하면 IllegalArgumentException이 발생한다
- 파라미터의 개수를 나타내는 더 구체적인 KFunction1, KFunction2 등을 사용할 수도 있다
```.kt
fun sum(x: Int, y: Int) = x + y

fun main() {
  val kFunction: KFunction2<Int, Int, Int> = ::sum
  println(kFunction.invoke(1, 2) + kFunction(3, 4)) // 10
  kFunction(1) // ERROR: No value passed for parameter p2
}
```
- KFunction에 대해 call이 아니라 invoke 메서드를 호출할때는 인자 개수나 타입을 틀릴 수 없다 (컴파일 X)
- KProperty도 call을 호출할 수 있으며 이는 getter를 호출한다
- 하지만 프로퍼티 인터페이스는 더 좋은 방법으로 get 메서드를 제공한다
- get 메서드를 사용하려면 프로퍼티가 선언된 방법에 따라 올바른 인터페이스를 사용해야 한다
- 최상위 읽기 전용과 가변 프로퍼티는 각각 KProperty0나 KMutableProperty0 인터페이스의 인스턴스로 표현되며
- 둘 다 인자가 없는 get 메서드를 제공한다
```.kt
var counter = 0
fun main() {
  val kProperty = ::counter
  kProperty.setter.call(21)
}
```
