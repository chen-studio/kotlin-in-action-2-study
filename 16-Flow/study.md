## 0.Objectives 
- 값의 연속적인 스트림을 모델링하는 플로우를 다루는 방법
- 콜드 플로우와 핫 플로우의 차이점과 사용 사례

## 1. 플로우는 연속적인 값의 스트림을 모델링한다
- 일시 중단 함수는 한 번 또는 여러번 실행을 중단할 수 있음
- 그러나 suspend 함수는 원시 타입, 객체, 객체의 컬렉션과 같은 단일값만 반환할 수 있음
- 이 예제는 createValues라는 일시 중단 함수를 사용해 3개의 값을 생성한다
```.kt
import kotlinx.coroutines.delay
import kotlinx.coroutines.runBlocking
import kotlin.time.Duration.Companion.seconds
 
suspend fun createValues(): List<Int> {
    return buildList {
        add(1)
        delay(1.seconds)
        add(2)
        delay(1.seconds)
        add(3)
        delay(1.seconds)
    }
}
 
fun main() = runBlocking {
    val list = createValues()
    list.forEach {
        log(it)
    }
}
 
// 3099 [main @coroutine#1] 1
// 3107 [main @coroutine#1] 2
// 3107 [main @coroutine#1] 3
```
- 예제에서 첫 번째 원소는 즉시 사용할 수 있었고, 2번째 원소는 1초뒤에 사용 가능했을 것이다
- 이처럼 함수가 여러 값을 시간이 지남에 따라 계산하는상황에서, 실행을 마칠 때까지 기다리지 않고 비동기적으로 값을 사용할 수 있도록 반환하고 싶을때 Flow가 유용하다
- 플로우는 반응형 스트림의 구현체로 RxJava에서 영감을 받았다

### 1.1 플로우를 사용하면 배출되지마자 원소를 처리할 수 있다
- 위 예시를 Flow를 사용해보자
```.kt
import kotlinx.coroutines.delay
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds
 
fun createValues(): Flow<Int> {
    return flow {
        emit(1)
        delay(1000.milliseconds)
        emit(2)
        delay(1000.milliseconds)
        emit(3)
        delay(1000.milliseconds)
    }
}
 
fun main() = runBlocking {
    val myFlowOfValues = createValues()
    myFlowOfValues.collect { log(it) }
}
 
// 29 [main @coroutine#1] 1
// 1100 [main @coroutine#1] 2
// 2156 [main @coroutine#1] 3
```
- 로그를 보면 원소가 배출되는 즉시 표시된다는 것을 알 수 있다

### 1.2 코루틴 플로우의 여러 유형
- 플로우는 콜드 플로우와 핫 플로우 2가지 카테고리로 나뉜다
- 콜드 플로우는 비동기 데이터 스트림으로 값이 실제로 소비되기 시작할 때만 값을 배출한다
- 핫 플로우는 값이 실제로 소비되고 있는지와 상관없이 독립적으로 배출하며, 브로드캐스트 방식으로 동작한다

## 2. 콜드 플로우
- 콜드 플로우 작업의 다양한 측면을 살펴보자

### 2.1 flow 빌더 함수를 사용해 콜드 플로우 생성 
- 새로운 콜드 플로우를 생성하는 것은 간단하다
- 컬렉션과 마찬가지로 빌더 함수가 있다
- 이 함수는 flow라 불린다
```.kt
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds
 
fun main() {
    val letters = flow {
        log("Emitting A!")
        emit("A")
        delay(200.milliseconds)
        log("Emitting B!")
        emit("B")
    }
}
```
- flow 빌더 함수의 안에서는 emit 함수를 통해 값을 제공하고 수집자가 해당 값을 처리할때까지 suspend 된다. 이를 비동기 return과 비슷하게 생각할 수 있다
- flow 내부는 suspend 내부이므로 delay와 같은 일시중단 함수를 호출할 수 있다
- 하지만 위 코드를 실행하면 아무런 출력도 나타나지 않는다는 점을 지적할 만하다.
- 이는 빌더 함수가 연속적인 값의 스트림을 표현하는 Flow<T>타입의 객체를 반환하기 때문이다
- 이 플로우는 처음 비활성 상태이며 최종 연산자가 호출 되어야만 빌더에서 정의된 계산이 시작된다
- flow 빌더 함수를 호출해도 실제 작업이 이루어지지 않기때문에 아래처럼 작성할 수 있다
```.kt
import kotlinx.coroutines.flow.*

fun getElementsFromNetwork(): Flow<String> {
   return flow {
       // suspending network call here
   }
}
```
- 빌더 안의 코드는 플로우가 수집될 때만 실행되므로 시퀀스와 마찬가지로 무한 플로우를 정의하고 반환해도 괜찮다
```.kt
val counterFlow = flow {
   var x = 0
   while (true) {
       emit(x++)
       delay(200.milliseconds)
   }
}
```
### 2.2 콜드 플로우는 수집되기 전까지 작업을 수행하지 않는다
- flow에 대해 collect함수를 호출하면 그 로직이 실행된다
- flow를 수집하는 코드를 collector라고 부른다
```.kt
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.milliseconds

val letters = flow {
   log("Emitting A!")
   emit("A")
   delay(200.milliseconds)
   log("Emitting B!")
   emit("B")
}

fun main() = runBlocking {
   letters.collect {
       log("Collecting $it")
       delay(500.milliseconds)
   }
}

// 27 [main @coroutine#1] Emitting A!
// 38 [main @coroutine#1] Collecting A
// 757 [main @coroutine#1] Emitting B!
// 757 [main @coroutine#1] Collecting B
```
- 출력의 타임스탬프를 보면 collector가 플로우의 로직을 실행하는 책임이 있다는 것을 알 수 있다
- 원소 A와 B 사이의 지연시간은 약 700이다

<img width="580" height="563" alt="image" src="https://github.com/user-attachments/assets/b5580beb-11ee-42c5-a6f6-bf86b3fb618e" />

- 시퀀스가 최종 연산자를 사용할 때마다 다시 평가되는 것처럼 콜드 플로우에서 collect를 여러 번 호출하면 그 코드가 여러 번 실행된다
```.kt
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds

fun main() = runBlocking {
   letters.collect {
       log("(1) Collecting $it")
       delay(500.milliseconds)
   }
   letters.collect {
       log("(2) Collecting $it")
       delay(500.milliseconds)
   }
}

// 23 [main @coroutine#1] Emitting A!
// 33 [main @coroutine#1] (1) Collecting A
// 761 [main @coroutine#1] Emitting B!
// 762 [main @coroutine#1] (1) Collecting B
// 1335 [main @coroutine#1] Emitting A!
// 1335 [main @coroutine#1] (2) Collecting A
// 2096 [main @coroutine#1] Emitting B!
// 2096 [main @coroutine#1] (2) Collecting B
```
- collect 함수는 플로우의 모든 원소가 처리될 때까지 일시 중단된다
- 그러나 플로우에 무한한 원소가 있을 수 있으므로 collect함수도 무기한 일시 중단될 수 있다
- 이런경우 플로우 수집을 취소할 수 있다

### 2.3 플로우 수집 취소
```.kt
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.seconds

fun main() = runBlocking {
   val collector = launch {
       counterFlow.collect {
           println(it)
       }
   }
   delay(5.seconds)
   collector.cancel()
}

// 1 2 3 ... 24
```

### 2.4 콜드 플로우의 내부 구현
- 콜드 플로우는 Flow와 FlowCollector라는 2가지 인터페이스만 필요하다
```.kt
interface Flow<out T> {
   suspend fun collect(collector: FlowCollector<T>)
}

interface FlowCollector<in T> {
   suspend fun emit(value: T)
}
```
- flow빌더 함수를 사용하여 플로우를 정의 할때 제공된 람다의 수신 객체 타입은 FlowCollector다
- 이 때문에 빌더 안에서 emit을 호출할 수 있다
- emit함수는 collect안에 전달된 람다를 호출하며 결과적으로 두 람다가 서로 호출하는 구조를 갖는다
1. collect를 호출하면 플로우 빌더 함수의 본문 실행
2. 이 코드가 emit을 호출하면 emit에 전달된 파라미터로 collect에 전달된 람다가 호출된다
3. 람다 표현식이 실행을 완료하면 함수는 빌더 함수의 본문으로 돌아가 계속 실행된다
```.kt
val letters = flow {
   delay(300.milliseconds)
   emit("A")
   delay(300.milliseconds)
   emit("B")
}

letters.collect { letter ->
   println(letter)
   delay(200.milliseconds)
}
```
<img width="828" height="593" alt="image" src="https://github.com/user-attachments/assets/d0ce1d27-147c-47ef-ad84-78a08f815ad2" />

### 2.5 채널 플로우를 사용한 동시성 플로우
- flow 빌더 함수를 사용해 만든 콜드 플로우는 모두 순차적으로 실행된다
- 만약 병렬로 수행하려면 어떻게 해야할까?
```.kt
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.*
import kotlin.random.Random
import kotlin.time.Duration.Companion.milliseconds

suspend fun getRandomNumber(): Int {
   delay(500.milliseconds)
   return Random.nextInt()
}

val randomNumbers = flow {
   repeat(10) {
       emit(getRandomNumber())
   }
}

fun main() = runBlocking {
   randomNumbers.collect {
       log(it)
   }
}

// 583 [main @coroutine#1] 1514439879
// 1120 [main @coroutine#1] 1785211458
// 1693 [main @coroutine#1] -996479986
// ...
// 5463 [main @coroutine#1] -2047597449
```
- 이 플로우를 수집하는데 약 5초가 걸리는데, getRandomNumber 호출이 순차적으로 실행되기 때문이다
<img width="1122" height="105" alt="image" src="https://github.com/user-attachments/assets/0866ee63-6f27-4ae5-a52d-04a460d11f15" />

- 만약 여기서 launch, async등을 사용하면 시간을 더 줄일 수 있지 않을까?
- 과연 아래처럼 했을때 동작이 수행될지 보자
```.kt
val randomNumbers = flow {
    coroutineScope {
        repeat(10) {
            launch { emit(getRandomNumber()) }
        }
    }
}
```
- 예상과 달리 이 코드를 수행하려고 하면 Flow invariant is violated: Emission from another coroutine is detected. FlowCollector is not thread-safe and concurrent emissions are prohibited. 라는 오류 메시지가 나타날 것이다
- 이는 기본적인 콜드 플로우 추상화가 같은 코루틴 안에서만 emit할 수 있도록 허용하기 때문이다
- 이런 상황에서 사용할 수 있는 플로우가 ChannelFlow이다. 
- 채널 플로우는 플로우의 특수한 형태이며 emit함수를 제공하지 않고 여러 코루틴에서 send를 사용해 값을 제공할 수 있다
```.kt
import kotlinx.coroutines.flow.channelFlow
import kotlinx.coroutines.launch
 
val randomNumbers = channelFlow {
    repeat(10) {
        launch {
            send(getRandomNumber())
        }
    }
}
```
- 채널 플로우를 앞의 코드와 마찬가지로 수집하면 getRandomNumber함수가 실제로 동시적으로 실행되며 전체 실행시간이 약 500밀리초로 줄어든다
```.kt
553 [main] -1927966915
568 [main] 222582016
...
569 [main] 1827898086
```
<img width="670" height="383" alt="image" src="https://github.com/user-attachments/assets/0c03f723-aaee-438d-8f46-ba4f44b5acbf" />

#### Flow vs ChannelFlow 어떤것을 선택해야 하는가?
- channelFlow를 사용하면 동시성 제어도 가능하며 flow의 모든 기능을 사용할 수 있다
- 하지만 내부적으로 Channel이라는 다른 동시성 요소를 사용하여 약간의 오버헤드가 있다
- 따라서 실제로 동시에 여러가지 작업을 수행하려고 할때만 ChannelFlow를 선택하는 것이 좋다

## 3. 핫 플로우
- 핫 플로우는 여러명의 구독자라고 불리는 수집자들이 배출된 항목을 공유한다.
- 핫 플로우는 항상 활성 상태이기 때문에 구독자의 유무에 관계없이 값을 발행할 수 있으며
- 수집자가 존재하는지 여부에 상관없이 값을 배출해야 하는 경우에 적합하다
- SharedFlow : 값을 브로드캐스트 하기 위해 사용
- StateFlow: 상태를 전달하는 특별한 경우에 사용

### 3.1 SharedFlow는 값을 구독자에게 브로드캐스트 한다
- SharedFlow는 구독자 존재 여부와 관계없이 배출이 발생하는 브로드 캐스트 방식으로 동작한다
<img width="750" height="431" alt="image" src="https://github.com/user-attachments/assets/50f33dc6-a653-423b-acd3-07dc6a56758a" />

- 코드를 보고 이해해보자
```.kt
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.random.*
import kotlin.time.Duration.Companion.milliseconds
 
class RadioStation {
    private val _messageFlow = MutableSharedFlow<Int>()
    val messageFlow = _messageFlow.asSharedFlow()
 
    fun beginBroadcasting(scope: CoroutineScope) {
        scope.launch {
            while(true) {
                delay(500.milliseconds)
                val number = Random.nextInt(0..10)
                log("Emitting $number!")
                _messageFlow.emit(number)
            }
        }
    }
}
```
- 코드를 보면 SharedFlow를 만드는 방식이 콜드 플로우와 다르다는 것을 알 수 있다
- 플로우 빌더를 사용하는 대신 가변적인 플로우에 대한 참조를 얻는다
- RadioStation클래스의 인스턴스를 생성하고 beginBroadcasting 함수를 호출하면 구독자가 없어도 브로드캐스트가 즉시 시작된다
```.kt
fun main() = runBlocking {
    RadioStation().beginBroadcasting(this)
}
 
// 575 [main @coroutine#2] Emitting 2!
// 1088 [main @coroutine#2] Emitting 10!
// 1593 [main @coroutine#2] Emitting 4!
// ...
```
- 구독자를 추가하는 방법은 마찬가지로 collect를 호출하면 된다
- 하지만 콜드 플로우와 달리 구독 시작 이후에 배출된 값만 수신할 수 있다
```.kt
fun main(): Unit = runBlocking {
   val radioStation = RadioStation()
   radioStation.beginBroadcasting(this)
   delay(600.milliseconds)
   radioStation.messageFlow.collect {
       log("A collecting $it!")
   }
}

611 [main @coroutine#2] Emitting 8!
1129 [main @coroutine#2] Emitting 9!
1131 [main @coroutine#1] A collecting 9!
1647 [main @coroutine#2] Emitting 1!
1647 [main @coroutine#1] A collecting 1!
```
- 아래처럼 두번째 구독자를 추가할 수도 있다
```.kt
launch {
   radioStation.messageFlow.collect {
       log("B collecting $it!")
   }
}
```
#### 구독자를 위한 값 replay
- MutableSharedFlow에는 replay라는 속성이 있다
- 이 값은 새 구독자를 위한 캐시 공간이며 이 값에 따라 새로운 구독자에게 replay 만큼의 캐시를 전달한다
```.kt
private val _messageFlow = MutableSharedFlow<Int>(replay = 5)
```
- 이렇게 바꾸고 나면 600밀리초가 지난 다음에 수집자를 시작하더라도 구독 직전에 발생한 최대 5개의 값을 수신할 수 있다

#### shareIn으로 콜드 플로우를 SharedFlow로 전환
```.kt
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.random.*
import kotlin.time.Duration.Companion.milliseconds

fun querySensor(): Int = Random.nextInt(-10..30)

fun getTemperatures(): Flow<Int> {
   return flow {
       while(true) {
           emit(querySensor())
           delay(500.milliseconds)
       }
   }
}
```
- 이 함수를 여러번 호출하려 한다면, 각 수집자가 센서에 독립적으로 질의를 하게 된다
```.kt
fun celsiusToFahrenheit(celsius: Int) =
    celsius * 9.0 / 5.0 + 32.0
 
fun main() {
    val temps = getTemperatures()
    runBlocking {
        launch {
            temps.collect {
                log("$it Celsius")
            }
        }
        launch {
            temps.collect {
                log("${celsiusToFahrenheit(it)} Fahrenheit")
            }
        }
    }
}
```
- 우리가 원하는 것은 이게 아닌 모두 같은 원소를 받기를 원한다
- shareIn 함수를 사용하면 주어진 콜드 플로우를 한 플로우인 공유 플로우로 변환할 수 있다
- 이 변환은 플로우의 코드가 실행되게 하므로 shareIn을 코루틴 안에서 호출해야 하여 CoroutineScope를 파라미터로 받는다
```.kt
fun main() {
   val temps = getTemperatures()
   runBlocking {
       val sharedTemps = temps.shareIn(this, SharingStarted.Lazily)
       launch {
           sharedTemps.collect {
               log("$it Celsius")
           }
       }
       launch {
           sharedTemps.collect {
               log("${celsiusToFahrenheit(it)} Fahrenheit")
           }
       }
   }
}

// 45 [main @coroutine#3] -10 Celsius
// 52 [main @coroutine#4] 14.0 Fahrenheit
// 599 [main @coroutine#3] 11 Celsius
// 599 [main @coroutine#4] 51.8 Fahrenheit
```
- shareIn의 두 번째 파라미터인 started는 플로우가 실제로 언제 시작되야 하는지 정의한다
- Eagerly는 플로우 수집을 즉시 시작
- Lazily는 첫 번째 구독자가 나타날 때 시작
- WhileSubscribed는 첫 번째 구독자가 나타나야 수집을 시작, 마지막 구독자가 사라지면 플로우 수집을 취소한다

### 3.2 시스템 상태 추적: StateFlow
- 동시 시스템에서 자주 요구되는 사항은 상태를 추적하는 것이다
- 코루틴은 이런 처리를 위해 특수화된 추상화인 StateFlow를 제공한다
- StateFlow와 관련해서 아래 사항들을 알아보자
- StateFlow 생성, 구독자에게 노출하는 방법
- StateFlow의 값을 안전하게 갱신하는 방법
- 값이 실제로 변경될 때만 StateFlow가 값을 배출하게 하는 동등성 기반 통합 개념
- 콜드 플로우를 StateFlow로 변환하는 방법
- 생성하는 방법은 SharedFlow와 거의 동일하다. 그리고 값을 배출하는 emit 대신 update 함수를 사용한다
```.kt
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.*
 
class ViewCounter {
    private val _counter = MutableStateFlow(0)
    val counter = _counter.asStateFlow()
 
    fun increment() {
        _counter.update { it + 1 }
    }
}
 
fun main() {
    val vc = ViewCounter()
    vc.increment()
    println(vc.counter.value)
    // 1
}
```
- MutableStateFlow는 value라는 속성을 가지고 있어 setValue, getValue 모두 가능하다
- 하지만 값 갱신은 value를 직접 set하는것 보단 update 함수를 사용하는 것이 좋다
- 아래 예시를 보자
```.kt
fun increment() {
   _counter.value++
}
```
- 이 증가 연산은 원자적이지 않고 동기화 문제를 가지고 있다
- update 함수를 사용하면 아래처럼 사용할 수 있다
```.kt
fun increment() {
    _counter.update { it++ }
}
```
- 참고 update
```.kt
/**
 * Updates the [MutableStateFlow.value] atomically using the specified [function] of its value.
 *
 * [function] may be evaluated multiple times, if [value] is being concurrently updated.
 */
public inline fun <T> MutableStateFlow<T>.update(function: (T) -> T) {
    while (true) {
        val prevValue = value
        val nextValue = function(prevValue)
        if (compareAndSet(prevValue, nextValue)) {
            return
        }
    }
}
```

#### StateFlow는 실제로 값이 달라졌을 때만 값을 배출한다: 동등성 기반 통합
- StateFlow에 setValue하거나 update를 할때, 이전 값과 동일하면 구독자에게 전달하지 않는다
<img width="602" height="640" alt="image" src="https://github.com/user-attachments/assets/0dde4602-e365-4aa8-b68a-80c75bd66dc6" />


#### stateIn으로 콜드 플로우를 StateFlow로 변환하기
- 콜드 플로우를 SharedFlow로 변환했던 것 처럼 stateIn을 사용하면 StateFlow로 변환할 수 있다
- 이렇게 하면 플로우에서 배출된 최신 값을 항상 읽을 수 있다
- SharedFlow와 마찬가지로 여러 수집자를 추가하거나 value속성에 접근해도 업스트림 플로우는 실행되지 않는다
```.kt
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds

fun main() {
   val temps = getTemperatures()
   runBlocking {
       val tempState = temps.stateIn(this)
       println(tempState.value)
       delay(800.milliseconds)
       println(tempState.value)
       // 18
       // -1
   }
}
```

### 3.3 StateFlow와 SharedFlow의 비교
- 정확하게 말하면 StateFlow는 SharedFlow의 특수한 형태이다
- SharedFlow의 replay = 1, bufferOverflow = DROP_OLDEST, distinctUntilChanged() 옵션을 주면 StateFlow와 동일한 효과를 낸다
- 두 코드를 비교해 보자

```.kt
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds

class Broadcaster {
   private val _messages = MutableSharedFlow<String>()
   val messages = _messages.asSharedFlow()
   fun beginBroadcasting(scope: CoroutineScope) {
       scope.launch {
           _messages.emit("Hello!")
           _messages.emit("Hi!")
           _messages.emit("Hola!")
       }
   }
}

fun main(): Unit = runBlocking {
   val broadcaster = Broadcaster()
   broadcaster.beginBroadcasting(this)
   delay(200.milliseconds)
   broadcaster.messages.collect {
       println("Message: $it")
   }
}

// No values are collected, nothing is printed
```

```.kt
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds

class Broadcaster {
   private val _messages = MutableStateFlow<List<String>>(emptyList())
   val messages = _messages.asStateFlow()
   fun beginBroadcasting(scope: CoroutineScope) {
       scope.launch {
           _messages.update { it + "Hello!" }
           _messages.update { it + "Hi!" }
           _messages.update { it + "Hola!" }
       }
   }
}

fun main() = runBlocking {
   val broadcaster = Broadcaster()
   broadcaster.beginBroadcasting(this)
   delay(200.milliseconds)
   println(broadcaster.messages.value)
}
// [Hello!, Hi!, Hola!]
// 마지막 값 전달 받음
```

### 3.4 Hot Flow, Cold Flow, SharedFlow, StateFlow 언제 어떤 플로우를 사용?
- 콜드 플로우
- 기본적으로 비활성
- 수집자 1명을 대상으로 값 발행
- 수집자는 모든 배출을 받음
- 보통은 완료됨
- 하나의 코루틴에서 배출 발생 (channelFlow 제외)
- 핫 플로우
- 기본적으로 활성
- 여러 구독자가 있음
- 구독자는 구독 시작 시점 이후부터 배출을 받음
- 완료되지 않음
- 여러 코루틴에서 배출될 수 있음
- 플로우를 처리할 수 있는 여러 연산자를 제공하는데 17장에서 이어서 다룬다
