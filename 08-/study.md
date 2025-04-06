## 0.Objectives
- 원시 타입과 다른 기본 타입 및 자바 타입과의 관계
- 코틀린 컬렉션과 배열 및 이들의 nullable과 상호운용성

## 1.원시 타입과 기본 타입

### 1-1 정수, 부동소수점 수, 문자, 불리언 값을 원시 타입으로 표현
- 자바는 primitive type과 reference type을 비교한다
- primitive type에는 그 값이 직접 들어가지만 reference type은 메모리상의 객체 위치가 들어간다
- primitive type은 값을 더 효율적으로 저장하고 여기저기 전달할 수 있지만 메서드 호출이나 컬렉션에 담는것은 불가능하다
- 자바는 primitive type에 대해 컬렉션을 정의하려면 java.lang.Interger과 같은 wrapper클래스를 사용해야 한다
- 코틀린은 이를 구분하지 않으며 Int만 존재한다
- 코틀린은 대부분의 경우는 자바의 int로 컴파일 되며 이런 컴파일이 불가능한 경우에만 java.lang.Interger 객체를 사용한다

### 1-2 양수를 표현하기 위해 모든 비트 범위 사용: 부호 없는 숫자 타입
- 양수를 표현하기 위해 모든 비트 범위를 사용하고 싶은 경우(비트와 바이트 수준의 작업, 비트맵 픽셀 조작, 2진데이터 등) 코틀린은 JVM의 일반적인 원시 타입을 확장해 부호 없는 타입을 제공한다
- UByte: 8비트, 0 ~ 255
- UShort: 16비트, 0 ~ 65535
- UInt: 32비트, 0 ~ 2^32-1
- ULong: 64비트, 0 ~ 2^64-1
- 부호 없는 숫자 타입들은 상응하는 부호 있는 타입의 범위를 '시프트'해서 같은 크기의 메모리를 사용해 더 큰 양수 범위를 표현할 수 있게 해준다
- 예를들어 Int는 대략 -20억 ~ 20억 사이의 값을 표현하지만 UInt는 0부터 대략 40억 까지의 값을 표현할 수 있다
- 이 타입도 필요할 때만 wrapping된다
- JVM자체에는 부호 없는 수에 대한 원시타입을 제공하지 않는다. 코틀린이 내부적으로 이를 처리하고 실제 값은 Int로 컴파일된다

### 1-3 nullable 기본 타입: Int? Boolean? 등
- 자바에서는 null을 reference 타입의 변수에만 대입할 수 있기 때문에 이 타입은 자바의 wrapper type으로 컴파일된다

### 1-4 수 변환
- 코틀린은 한 타입의 수를 다른 타입의 수로 자동 변환하지 않는다
- 결과 타입이 허용하는 수의 범위가 원래 타입의 범위보다 넓은 경우조차도 자동 변환은 불가능하다
```.kt
val i = 1
val l: Long = i // Error: type mismatch
```
- 코틀린은 모든 원시 타입에 대해 toByte(), toShort(), toChar()등 변환 함수를 제공하며 양방향으로 모두 제공된다
- Int.toLong(), Long.toInt() 등
- 자바에서 reference 타입을 비교하는 경우 new Integer(42).equals(new Long(42))는 false다
- 만약 코틀린에서 암시적 변환이 허용된다면 아래와 같이 쓸 수 있을 것이다
```.kt
val x = 1
val list = listOf(1L, 2L, 3L)
x in list
```
- 하지만 이 구문은 허용되지 않으며 아래처럼 x를 toLong()으로 변환해야 한다
```.kt
val x = 1
x.toLong() in listOf(1L, 2L, 3L)
```
- 코틀린은 소스코드에서 단순한 10진수(정수) 외에 아래와 같은 수 리터럴을 허용한다
1. L 접미사가 붙은 Long 타입 : 123L
2. 표준 부동소수점 표기법을 사용한 Double 타입 리터럴: 0.12, 2.0, 1.2e10, 1.2e-10 등
3. f나 F 접미사가 붙은 Float 타입 리터럴: 123.4f, .456f, 1e3f
4. 0x나 0X 접두사가 붙은 16진수 리터럴; 0xCAFEBABE, 0xbcdL
5. 0b나 0B 접두사가 붙은 2진 리터럴: 0b00000101
6. U 접미사가 붙은 부호 없는 정수 리터럴: 123U, 123UL, 0x10cU 등
- 문자 리터럴의 경우 자바와 마찬가지 구문을 사용한다
- 작은따오묘 안에 문자를 넣으면 되고, 이스케이프 시퀀스를 사용할 수도 있다. `1`. `\t` `\u009`(유니코드 이스케이프 시퀀스로 정의한 탭 문자)
- 숫자 리터럴을 사용할때는 보통 변환 함수를 호출할 필요가 없다(42L, 42.0f 등)
```.kt
val b: Byte = 1 <- 상수 값은 적절한 타입으로 해석됨
val l = b + 1L <- +는 Byte와 Long을 인자로 받을 수 있다
printALong(42) <- 컴파일러는 42를 Long 값으로 해석한다
```
- 코틀린 산술 연산자에서도 자바와 똑같이 숫자 연산 시 overflow나 underflow가 발생할 수 있다. 코틀린은 이들을 검사하느라 추가 비용을 들이지 않음
```.kt
println(Int.MAX_VALUE + 1)
// -2147483648 <- 오버플로로 인해 값이 양수 최댓값에서 음수 최솟값으로 넘어감
println(Int.MIN_VALUE - 1)
// 2147483647 <- 오버플로로 인해 값이 음수 최솟값에서 양수 최댓값으로 넘어감
```

- 코틀린 표준 라이브러리는 문자열을 원시 타입으로 변환하는 여러 함수를 제공한다
- 문자열 내용을 각 타입으로 파싱하려 시도하며 파싱에 실패하면 NumberFormatException이 발생한다
```.kt
"42".toInt() // 42
```
- 파싱에 실패하면 Exception 대신 null을 반환하는 *OrNull() 함수도 존재한다
```.kt
"seven".toIntOrNull() // null
```
- toBoolean 함수는 인자로 받은 문자열이 null이 아니고 문자열의 내용이 단어 true와 같으면(대소문자 구분없음) true를 반환한다. 그 이외엔 false
```.kt
"trUE".toBoolean // true
"7".toBoolean // false
null.toBoolean // false
```
- 정확히 true나 false와 일치시키고 싶은 경우 toBooleanStrict를 사용하면 true, false이외에는 예외를 던진다

### 1-5 Any와 Any?: 코틀린 타입 계층의 뿌리
- 자바에서는 reference타입의 조상 타입이 Object이고 primitive 타입은 그런 계층에 포함되지 않는다
- 하지만 코틀린에서는 모든 타입의 조상 타입이 Any 타입이다
- 만약 primitive type값을 Any 타입의 변수에 대입하면 자동으로 값을 객체로 감싼다
```.kt
val answer: Any = 42 // Boxing된다 java.lang.Integer
```
- Any타입은 java.lang.Object에 대응한다
- 만약 java.lang.Object에만 존재하는 wait이나 notify같은 메서드를 사용하려면 java.lang.Object 타입으로 캐스팅해야한다
- Any타입은 null이 될 수 없으며 null을 포함하려면 Any?로 선언해야한다

### 1-6 Unit: 코틀린의 void
- 코틀린 Unit 타입은 자바 void와 같은 기능을 한다
- 반환 타입을 명시하지 않는 경우 기본으로 Unit타입이 된다
```.kt
fun f(): Unit { /*...*/ }
fun f() { /*...*/}
```
- 코틀린의 Unit이 자바void와 다른 점은 Unit을 타입 인자로 쓸 수 있다
```.kt
interface Processor<T> {
    fun process(): T
}

class NoResultProcessor : Processor<Unit> {
    override fun process() { // Unit을 반환하지만 지정 필요 X
        // Something
    }
}
```
- Unit을 반환하지만 return Unit을 명시하지 않아도 되는 이유는 컴파일러가 이를 처리해주기 때문이다
- Unit이라는 이름은 함수형 프로그래밍에서 단 하나의 인스턴스만 갖는 타입을 의미하기 위함이다

### 1-7 Nothing 타입: 이 함수는 결코 반환되지 않는다
- 코틀린에는 결코 성공적으로 값을 돌려주는 일이 없으므로 `반환값`이라는 개념 자체가 의미 없는 함수가 일부 존재한다
- 이런 경우 Nothing이라는 특별한 반환 타입을 사용한다
```.kt
fun fail(message: String): Nothing {
    throw IllegalStateException(message)
}
```
- Nothing 타입은 아무 값도 포함하지 않는다. 따라서 Nothing은 함수의 반환 타입이나 반환 타입으로 쓰일 타입 파라미터로만 쓸 수 있다
- 그 외에는 사용하더라도 그 변수에 아무 값도 저장할 수 없으므로 아무런 의미도 없다
- 컴파일러는 Nothing 반환 타입인 함수가 결코 정상 종료되지 않음을 알고 그 함수를 호출하는 코드를 분석한다
```.kt
val address = company.address ?: fail("No address)
println(address.city)
```
- 엘비스 연산자의 오른쪽에서 예외가 발생한다는 사실을 컴파일러가 파악하고 address의 값이 null이 아님을 추론할 수 있다

## 2. 컬렉션과 배열
- 자바와 코틀린의 컬렉션간의 관계에 대해 알아보자

### 2-1. 널이 될 수 있는 값의 컬렉션과 널이 될 수 있는 컬렉션
```.kt
fun readNumbers(text: String): List<Int?> {
    val result = mutableListOf<Int?>()
    for (line in text.lineSequence()) {
        val numberOrNull = line.toIntOrNull()
        result.add(numberOrNull)
    }
    return result
}
```
- List<Int?> 는 Int? 타입의 값을 저장할 수 있다
- List<Int>? 와 List<Int?> 의 차이에 대해 잘 이해해야 한다
- List<Int>? 는 객체 자체가 null이 될 수 있고 내부 element는 null이 될 수 없다
- List<Int?> 는 객체 자체는 null이 될 수 없지만 내부 element는 null이 될 수 있다
- 앞에서 배운 람다에 대한 지식을 이용하면 위 함수를 더 단축시킬 수 있다
```.kt
fun readNumbers2(text: String): List<Int?> = 
    text.lineSequence().map { it.toIntOrNull() }.toList()
```
- 경우에 따라 List자체도 null이 될 수 있고 내부 element도 nullable인 경우 List<Int?>? 로 나타낼 수 있다
- 그리고 null이 아닌 요소만으로 구성된 리스트를 만들고 싶은 경우 filterNotNull()을 사용하면 좋다
```.kt
fun addValidNumbers(numbers: List<Int?>) {
    val validNumbers = numbers.filterNotNull()
    ...
}
```
- 또한 filterNotNull의 결과는 List<T?> -> List<T> 이다

### 2-2 읽기 전용과 변경 가능한 컬렉션
- 코틀린의 컬렉션의 특징 중 하나는 변경 가능한 컬렉션(MutableCollection)과 읽기 전용 컬렉션(Collection)을 구분 했다는 점이다
- Collelction 인터페이스는 읽기 전용이며 size(), iterator(), contains() 등을 제공하며
- MutableCollection 인터페이스는 Collection 인터페이스를 확장하여 add(), remove(), clear() 등의 메서드를 제공한다
```.kt
fun <T> copyElements(source: Collection<T>, target: MutableCollection<T>) {
    for (item in source) {
        target.add(item)
    }
}

fun main() {
    val source: Collection<Int> = arrayListOf(3, 5, 7)
    val target: MutableCollection<Int> = arrayListOf(1)
    copyElements(source, target)
    println(target)
    [1, 3, 5, 7]
}
```
- 객체가 실제로는 변경 가능한 컬렉션이라 하더라도 target에 해당하는 인자로 읽기 전용 컬렉션 타입의 값을 넘길 수 없다
```.kt
fun main() {
    val source: Collection<Int> = arrayListOf(3, 5 ,7)
    val target: Collection<Int> = arrayListOf(1)
    copyElements(source, target) <- 컴파일 에러
}
```
- 하지만 읽기 전용 컬렉션이라 하더라도 항상 쓰레드 안전하지는 않다. 읽기 전용 컬렉션과 변경 가능한 컬렉션이 같은 값을 참조할 수 있고 이 경우 값이 변경될 수 있다
- 여러 쓰레드에서 컬렉션을 변경하려고 시도하는 경우 ConcurrentModificationException이나 다른 오류가 발생할 가능성이 있다

### 2-3 코틀린 컬렉션과 자바 컬렉션은 밀접히 연관됨
- 컬렉션 타입: List / 읽기 전용 타입: listOf, List / 변경 가능 타입: mutableListOf, MutableList, arrayListOf, buildList
- 컬렉션 타입: Set / 읽기 전용 타입: setOf / mutableSetOf, hashSetOf, linkedSetOf, sortedSetOf, buildSet
- 컬렉션 타입: Map / 읽기 전용 타입: mapOf / mutableMapOf, hashMapOf, linkedMapOf, sortedMapOf, buildMap
- setOf와 mapOf는 Set과 Map의 읽기 전용 인터페이스의 인스턴스를 반환하지만 내부적으로는 변경 가능한 클래스이다
- 하지만 그 둘이 변경 가능한 클래스라는 사실에 의존하면 안 된다
- 미래에는 setOf나 mapOf가 진정한 불변 컬렉션 인스턴스를 반환하게 될 수도 있다
- 자바 메서드를 호출하되 컬렉션을 인자로 넘겨야 한다면 Collection 또는 MutableCollection 모두 인자로 넘길 수 있다
- 다만 자바 메서드에 읽기전용 Collection을 넘겨도 자바에서는 값을 수정할 수 있다. 그 책임은 개발자에게 있다

### 2-4 자바에서 선언한 컬렉션은 코틀린에서 플랫폼 타임으로 보임
- 일반적인 상황에선 문제가 발생할 가능성이 적지만 컬렉션 타입이 시그니처에 들어간 자바 메서드 구현을 오버라이드 하는 경우 여러 고려사항이 생긴다
1. 컬렉션이 null이 될 수 있는지?
2. 컬렉션의 원소가 null이 될 수 있는지?
3. 컬렉션을 변경할 수 있는지?
```.java
interface FileContentProcessor {
    void processContents(
        File path,
        byte[] binaryContents,
        List<String> textContents
    );
}
```
- 이 인터페이스를 코틀린으로 구현하려면
1. 일부 파일이 이진 파일이고 이진 파일의 내용은 텍스트로 표현할 수 없는 경우가 있으므로 리스트는 null이 될 수 있다
2. 파일의 각 줄은 null일 수 없으므로 이 리스트의 원소는 null이 될 수 없다
3. 이 리스트는 파일의 내용을 표현하며 그 내용을 바꿀 필요가 없으므로 읽기 전용이다
- 이와같은 사항을 고려하하여 구현체를 작성하면 아래와 같이 구현할 수 있다
```.kt
class FileIndexer: FileContentProcessor {
    override fun processContents(
        path: File,
        binaryConents: ByteArray?,
        textContents: List<String>?
    )
}
```
- 다른 예시도 있지만 그닥 좋은 예시는 아니어서 생략

### 2-5 성능과 상호운용을 위해 객체의 배열이나 원시 타입의 배열을 만들기
- 코틀린 배열은 타입 파라미터를 받는 클래스다
- 배열의 원소 타입은 바로 그 타입 파라미터에 의해 정해진다
- 코틀린에서 배열을 만드는 방법은
1. arrayOf
2. arrayOfNulls
3. Array 
- 가 있다
```.kt
val letters = Array<String>(26) { i -> ('a'+i).toString }
val letters = Array(26) { i -> ('a'+i).toString } // 타입 추론으로 타입 인자 생략 가능
letters.joinToString(") // a - z 까지 알파벳 배열 
```
- 이런식으로 람다에서 값을 초기화하는 방법은 List, MutableList도 제공한다
- 컬렉션을 배열로 바꾸는 방법은 toTypedArray()를 사용할 수 있다
```.kt
val strings = listOf("a", "b", "c")
println("%s%s%s.format(*strings.toTypedArray()))
// format은 파라미터로 varargs (여러개의 파라미터를 넘긴다는 의미)를 사용함
// 먼저 List<String>을 Array<String>으로 변환하기 위해 toTypedArray() 사용
// *(스프레드 연산자)를 사용하면 Array에 담긴 값을 펴서 전달할 수 있음 
```
- 다른 제네릭에서처럼 배열 타입의 타입 인자도 항상 객체 타입이 된다. 
- Array<Int> 는 java.lang.Integer 배열이 된다
- 이런 불필요한 reference type 사용 방지를 위해 IntArray, ByteArray 등 원시 타입 배열을 사용하면 int[], byte[]로 컴파일된다
- 아직 안정화 되지 않았지만 부호 없는 값에 대한 배열인 UByteArray, UIntArray 등도 있다
