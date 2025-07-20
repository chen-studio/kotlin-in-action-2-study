## 0. Objectives
- 플로우를 변형하고 다루기 위한 연산자
- 중간 연산자와 최종 연산자
- 커스텀 플로우 연산자 

## 1. 플로우 연산자로 플로우 조작
- 플로우를 변환할때도 컬렉션의 연산자와 비슷한 연산자를 사용할 수 있음
<img width="530" height="318" alt="image" src="https://github.com/user-attachments/assets/707ef270-f76f-49b3-a360-81b793b19334" />


## 2. 중간 연산자는 업스트림 플로우에 적용되고 다운스트림 플로우를 반환한다
- 중간 연산자는 플로우에 적용돼 새로운 플로우를 반환
- 중간 연산자를 기준으로 업스트림과 다운스트림으로 구분
- 중간 연산자가 적용되는 플로우를 업스트림 플로우, 중간 연산자가 반환하는 플로우를 다운스트림 플로우라고 부른다
- 중간 연산자도 시퀀스와 마찬가지로 collect 하지 않으면 수행되지 않음
<img width="1031" height="476" alt="image" src="https://github.com/user-attachments/assets/2e57d04b-9e95-4b5d-8e49-15e3b921ca7c" />

- 시퀀스에서 사용할 수 있는 map, filter, onEach 등 모두 사용 가능
- 다만 시퀀스나 컬렉션의 원소가 아니라 플로우의 원소에 대해 작용

### 2-1. 업스트림 원소별로 임의의 값을 배출: transform 함수
```.kt
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() {
   val names = flow {
       emit("Jo")
       emit("May")
       emit("Sue")
   }
   val uppercasedNames = names.map {
       it.uppercase()
   }
   runBlocking {
       uppercasedNames.collect { print("$it ")}
   }
   // JO MAY SUE
}
```
- 여기서 각 이름을 대문자로 변형한 값 뿐 아니라 소문자 변형도 함께 배출하고 싶을 수 있다
- transform은 업스트림 플로우의 각 원소에 대해 원하는 만큼의 원소를 다운스트림 플로우에 배출할 수 있게 해준다
```.kt
import kotlinx.coroutines.flow.*

fun main() {
   val names = flow {
       emit("Jo")
       emit("May")
       emit("Sue")
   }
   // 또는 flowOf("Jo", "May", "Sue")도 가능

   val upperAndLowercasedNames = names.transform {
       emit(it.uppercase())
       emit(it.lowercase())
   }
   runBlocking {
       upperAndLowercasedNames.collect { print("$it ")}
   }
   // JO jo MAY may SUE sue
}
```

### 2-2. take나 관련 연산자는 플로우를 취소할 수 있다
- take(5)를 호출하면 원소가 5번 배출된 후 업스트림 플로우가 취소됨
```.kt
    val temps = getTemperatures()
    temps
        .take(5)
        .collect {
            log(it)
        }
}
 
// 37 [main @coroutine#1] 7
// 568 [main @coroutine#1] 9
// 1123 [main @coroutine#1] 2
// 1640 [main @coroutine#1] -6
// 2148 [main @coroutine#1] 7
```
- take 함수는 collector와 관련된 코루틴 스코프를 취소하는 방식 외에 플로우 수집을 제어된 방식으로 취소하는 또 다른 방법이다

### 2-3. 플로우의 각 단계 후킹: onStart, onEach, onCompletion, onEmpty
- 위 예시에서 플로우가 다섯 원소를 수집한 후 종료되는 것을 확인하기 위해 onCompletion 연산자를 사용할 수 있다
- 이 연산자는 플로우가 정상 종료되거나 취소되거나 예외로 종료된 후에 호출되는 람다를 지정할 수 있게 해준다
```.kt
fun main() = runBlocking {
   val temps = getTemperatures()
   temps
       .take(5)
       .onCompletion { cause ->
           if (cause != null) {
               println("An error occurred! $cause")
           } else {
               println("Completed!")
           }
       }
       .collect {
           println(it)
       }
}
```
- onStart는 플로우 수집이 시작될 때 첫 번째 배출이 일어나기 직전에 호출된다
- onEach는 업스트림 플로우에서 배출된 각 원소에 대해 작업을 수행한 후 이를 다운스트림 플로우에 전달한다
- 원소를 배출하지 않고 종료되는 플로우의 경우 orEmpty로 로직을 추가로 수행하면서 기본값을 제공할 수 있다
```.kt
flow
    .onEmpty {
        println("Nothing - emitting default value!")
        emit(0)
    }
    .onStart {
        println("Starting!")
    }
    .onEach {
        println("On $it!")
    }
    .onCompletion {
        println("Done!")
    }
    .collect()
```
```.kt
fun main() {
   runBlocking {
       process(flowOf(1, 2, 3))
       // Starting!
       // On 1!
       // On 2!
       // On 3!
       // Done!
       process(flowOf())
       // Starting!
       // Nothing - emitting default value!
       // On 0!
       // Done!
   }
}
```

### 2-4. 다운스트림 연산자와 수집자를 위한 원소 버퍼링: buffer 연산자
- 플로우 내부에서 많은 동작을 수행하게 될때 플로우의 원소를 수집하거나 onEach와 같은 연산자로 처리할때 suspend 함수를 호출하는 경우가 많다
```.kt
fun getAllUserIds(): Flow<Int> {
   return flow {
       repeat(3) {
           delay(200.milliseconds) // Database latency
           log("Emitting!")
           emit(it)
       }
   }
}

suspend fun getProfileFromNetwork(id: Int): String {
   delay(2.seconds) // Network latency
   return "Profile[$id]"
}
```
- 기본적으로 콜드플로우의 값 생산자는 수집자가 이전 원소를 처리할 때 까지 작업을 중단한다
```.kt
fun main() {
   val ids = getAllUserIds()
   runBlocking {
       ids
           .map { getProfileFromNetwork(it) }
           .collect { log("Got $it") }
   }
}

// 310 [main @coroutine#1] Emitting!
// 2402 [main @coroutine#1] Got Profile[0]
// 2661 [main @coroutine#1] Emitting!
// 4732 [main @coroutine#1] Got Profile[1]
// 5007 [main @coroutine#1] Emitting!
// 7048 [main @coroutine#1] Got Profile[2]
```
<img width="817" height="740" alt="image" src="https://github.com/user-attachments/assets/5e10cb2a-8455-45a6-b552-2373dcceec39" />
- 이처럼 생산자와 소비자의 속도 차이가 날때, 생산자가 suspend되지 않고 데이터를 쌓을 공간을 제공하는것이 buffer 연산자의 역할이다
```.kt
fun main() {
   val ids = getAllUserIds()
   runBlocking {
       ids
           .buffer(3)
           .map { getProfileFromNetwork(it) }
           .collect { log("Got $it") }
   }
}

// 304 [main @coroutine#2] Emitting!
// 525 [main @coroutine#2] Emitting!
// 796 [main @coroutine#2] Emitting!
// 2373 [main @coroutine#1] Got Profile[0]
// 4388 [main @coroutine#1] Got Profile[1]
// 6461 [main @coroutine#1] Got Profile[2]
```
<img width="1122" height="810" alt="image" src="https://github.com/user-attachments/assets/efa37a23-23af-498b-9911-35c3d0b08609" />


### 2-5. 중간값을 버리는 연산자: conflate 연산자
- 값 생산자가 방해받지 않고 작업을 계속할 수 있게 하는 또다른 방법은 수집자가 바쁜 동안 배출된 항목을 그냥 버리는 것이다
- conflate 연산자를 통해 이렇게 할 수 있다
```.kt
runBlocking {
    val temps = getTemperatures()
    temps
        .onEach {
            log("Read $it from sensor")
        }
        .conflate()
        .collect {
            log("Collected $it")
            delay(1.seconds)
        }
}

// 43 [main @coroutine#2] Read 20 from sensor
// 51 [main @coroutine#1] Collected 20
// 558 [main @coroutine#2] Read -10 from sensor
// 1078 [main @coroutine#2] Read 3 from sensor
// 1294 [main @coroutine#1] Collected 3
// 1579 [main @coroutine#2] Read 13 from sensor
// 2153 [main @coroutine#2] Read 26 from sensor
// 2556 [main @coroutine#1] Collected 26
```
- collector가 downstream의 동작을 수행중일때 데이터가 emit 되면, 그 값을 무시하게 된다

### 2-6 일정 시간 동안 값을 필터링 하는 연산자: debounce 연산자
- 어떤 상황에서는 플로우의 값을 처리하기 전에 잠시 기다리는 것이 유용할 수 있다
- 사용자가 검색 질의 문자열을 타이핑한다고 할 때 상황을 가정해보자
```.kt
val searchQuery = flow {
   emit("K")
   delay(100.milliseconds)
   emit("Ko")
   delay(200.milliseconds)
   emit("Kotl")
   delay(500.milliseconds)
   emit("Kotlin")
}
```

- 애플리케이션은 검색 버튼을 누르지 않아도 입력한 결과를 즉시 볼 수 있는 즉시 검색 기능을 제공하려 한다
- 이 플로우를 별도의 처리 없이 수집하는 각 키 입력마다 새로운 검색 요청이 시작된다
- 사용자가 타이핑하는 속도를 생각할때 매 글자마다 api요청을 보내면 매우 비효율적이다.
- 따라서 검색 동작을 수행하기 전 잠시 기다리는 것이다
- 250 milliseconds 만큼 기다렸다가 동작을 수행하는 flow를 작성해보자
```.kt
fun main() = runBlocking {
   searchQuery
       .debounce(250.milliseconds)
       .collect {
           log("Searching for $it")
       }
}

// 644 [main @coroutine#1] Searching for Kotl
// 876 [main @coroutine#1] Searching for Kotlin
```

### 2-7 플로우가 실행되는 코루틴 콘텍스트를 바꾸기: flowOn 연산자
- 플로우 연산자가 블로킹 I/O를 사용하거나 UI 스레드에서 작업할 필요가 있을 때, 일반 코루틴과 마찬가지로 디스패처(코루틴 콘텍스트)를 변경할 수 있다

```.kt
    runBlocking {
       flowOf(1)
           .onEach { log("A") }
           .flowOn(Dispatchers.Default)
           .onEach { log("B") }
           .flowOn(Dispatchers.IO)
           .onEach { log("C") }
           .collect()
   }
}

// 36 [DefaultDispatcher-worker-3 @coroutine#3] A
// 44 [DefaultDispatcher-worker-1 @coroutine#2] B
// 44 [main @coroutine#1] C
```

## 3. 커스텀 중간 연산자 만들기
- 직접 커스텀 중간 연산자는 어떻게 만들까?
- 예를들어 Double로 이루어진 플로우에서 마지막 n개 원소의 평균을 계산하는 연산자를 만들어 보자
```.kt
fun Flow<Double>.averageOfLast(n: Int): Flow<Double> =
   flow {
       val numbers = mutableListOf<Double>()
       collect {
           if (numbers.size >= n) {
               numbers.removeFirst()
           }
           numbers.add(it)
           emit(numbers.average())
       }
   }
```
```.kt
fun main() = runBlocking {
   flowOf(1.0, 2.0, 30.0, 121.0)
       .averageOfLast(3)
       .collect {
           print("$it ")
       }
}
// 1.0 1.5 11.0 51.0
```

## 4. 최종 연산자는 업스트림 플로우를 실행하고 값을 계산한다
- 플로우의 실행은 최종 연산자가 담당하며 최종 연산자는 단일 값이나 컬렉션을 계산하거나
- 플로우의 실행을 촉발시켜 지정된 연산과 부수 효과를 수행한다
- 가장 대표적인 최종 연산자는 collect이다
```.kt
fun main() = runBlocking {
   getTemperatures()
       .onEach {
           log(it)
       }
       .collect()
}
// or
fun main() = runBlocking {
   getTemperatures()
       .collect {
           log(it)
       }
}
```

- collect는 전체 플로우가 수집될 때까지 일시 중단되지만
- first나 firstOrNull같은 다른 최종 연산자는 원소를 받은 다음에 업스트림 플로우를 취소할 수 있다
- 예를들어 온도 센서에서 값을 하나만 얻고 싶다면
```.kt
fun main() = runBlocking {
   getTemperatures()
       .first()
}
```

### 4-1. 프레임워크는 커스텀 연산자를 제공한다
```.kt
@Composable fun TemperatureDisplay(temps: Flow<Int>) {
   val temperature = temps.collectAsState(null)
   Box {
       temperature.value?.let {
           Text("The current temperature is $it!")
       }
   }
}
```
