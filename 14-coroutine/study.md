## 0.Objectives
- 동시성과 병렬성의 개념에 대한 이해
- 코틀린에서 동시성 연산을 만드는 빌딩 블록, suspend 함수
- 코틀린에서 코루틴을 활용해 동시성 프로그래밍에 접근하는 방법

## 1. 동시성(Concurrency) vs 병렬성(Parallelism)
- 동시성: 여러 작업을 동시에 실행하는것을 의미하지만 모든 작업을 함께 실행하는 것은 아님
- n개의 작업의 일부분을 번갈아가면서 하나의 작업자가 수행하는 것도 동시성
- 즉, 코트를 여러 부분으로 나눠서 동시에 수행할 수 있는 능력을 의미
- 병렬성: 여러 작업을 동시에 실행하지만 실제로 물리적으로 동시에 실행하는 것을 의미

## 2. 코틀린의 동시성 처리 방법: 일시 중단 함수와 코루틴
- 코루틴은 코틀린의 가장 강력한 특징으로, 비동기적으로 실행되는 넌블로킹 동시성 코드를 우아하게 작성할 수 있게 해준다
- 스레드와 같은 전통적인 방식보다 훨씬 가볍게 동작한다
- 구조화된 동시성을 통해 동시성 작업과 그 생명주기를 관리할 수 있는 기능도 제공한다
- 먼저 스레드와 코루틴을 비교해본다

## 3. 스레드와 코루틴의 비교
- JVM에서 병렬 프로그래밍과 동시성 프로그래밍을 위한 고전적인 추상화는 스레드를 사용하는 것이다
- 스레드는 서로 독립적으로 동시에 실행되는 코드 블록을 지정할 수 있게 해준다
- 스레드도 코틀린에서 자바와 100% 호환된다
- 아래는 코틀린에서 스레드를 수행하는 예시를 보여준다
```.kt
import kotlin.concurrent.thread

fun main() {
  println("I'm on ${Thread.currentThread().name}")
  thread {
    println("And I'm on ${Thread.currentThread().name}")
  }
}
```
- 하지만 시스템 스레드를 생성하고 관리하는 것은 비용이 많이 든다
- 최신 시스템이더라도 한 번에 몇 천개의 스레드만 효과적으로 관리할 수 있다
- 어쨌든 스레드 관리의 비용이 많이 든다는 내용 설명
- 게다가 스레드가 네트워크 요청 등이 완료되기를 기다리는 동안에는 블록되며 그 동안에 해당 스레드는 아무런 작업을 할 수 없다
- 따라서 새 스레드를 생성할 떄는 매우 신중해야 하며 짧은 시간 동안 잠깐 사용하는 것은 피하는 것이 좋다
- 스레드는 기본적으로 독립적인 프로세스로 존재하기 때문에 작업을 관리하고 조정하는 데 어려움이 있을 수 있다
- 특히 취소나 예외 처리 같은 처리가 어렵기 떄문에 많은 제약이 존재한다

### 코틀린의 대안: 코루틴
- 코틀린에서 스레드의 대안으로 도입한 코루틴이라는 추상화의 장점에 대해 먼저 알아본다
- 코루틴은 초경량 추상화이며 일반적인 노트북에서도 100,000개 이상의 코루틴을 쉽게 실행할 수 있다
- 또한 코루틴은 생성하고 관리하는 비용이 저렴하다. 이는 훨씬 세밀한 작업이나 아주 짧은 시간 동안만 실행하는 작업에도 더 넓게 활용할 수 있다는 의미이다
- 코루틴은 시스템 자원을 블록시키지 않고 실행을 일시 중단할 수 있으며 나중에 중단된 지점에서 실행을 재개할 수 있다
- 이는 코루틴이 네트워크 요청이나 I/O 작업에 블로킹 스레드보다 훨씬 효율적이라는 의미다
- 코루틴은 구조화된 동시성 이라는 개념을 통해 동시 작업의 구조와 계층을 확립하며 취소 및 오류 처리를 위한 매커니즘을 제공한다
- 동시 계산의 일부가 실패하거나 더 이상 필요하지 않게 됐을 때 구조화된 동시성은 자식으로 시작된 다른 코루틴들도 함께 취소되도록 보장한다

### 실행
- 내부적으로 코루틴은 하나 이상의 JVM스레드에서 실행된다
- 이는 코루틴을 사용해 작성한 코드도 여전히 기본 스레드 모델이 제공하는 병렬성을 활용할 수 있지만,
- 운영체제가 부과하는 스레드의 한계에 얽매이지 않는다는 것을 의미한다

### 코루틴과 프로젝트 룸
- Project Loom은 JVM에 가상 스레드 형태의 경량 동시성을 도입해 JVM 스레드와 운영체제 스레드 간의 비용이 많이 드는 1:1 결합을 해소하기 위한 노력이다
- Project Loom이 도입한 Virtual Thread는 코루틴과 비슷한 문제를 해결하려고 노력하기 때문에 그 관계를 탐구할 가치가 있다
- Virtual Thread에 대해 잘 알고있는 서버 개발자분들의 의견을 구해요 ;-;

## 4. suspend function (일시중단 함수)
- 코루틴이 스레드, 반응형 스트림, 콜백과 같은 다른 동시성 접근 방식과 다른 핵심 속성으로는 상당수의 경우 코드 `형태`를 크게 변경하지 않아도 된다는 것이다
- 코드는 여전히 순차적으로 보이며 suspend 키워드가 어떻게 이런 방식을 가능하게 하는지 자세히 살펴보자

### 4-1 suspend 함수를 사용한 코드는 순차적으로 보인다
- 일반적인 로직을 보고 suspend 함수가 이를 어떻게 개선할 수 있는지 살펴보자
- showUserInfo라는 함수는 네트워크에서 정보를 요청하고, 정보를 사용자에게 보여준다
- login과 loadUserData 함수는 네트워크 요청을 보낸다.
- 이 코드는 현재 어떤 형태의 동시성도 사용하지 않고 단순히 하나의 함수가 완료된 후 다음 함수를 호출하며 마지막으로 네트워크 요청에 대한 응답이 오면 값을 반환한다
```.kt
fun login(credentials: Credentials): UserID
fun loadUserData(userID: UserID): UserData
fun showData(data: UserData)

fun showUserInfo(credentials: Credentials) {
  val userID = login(credentials)
  val userData = loadUserData(userID)
  showData(userData)
}
```
- 이 작업은 대부분 네트워크 작업 결과를 기다리는 데 소모하며, showUserInfo 함수가 실행중인 스레드를 블록시킨다.
- 앞에서 언급한 것처럼 스레드를 블록하는 것은 바람직 하지 않다
- 이 함수를 코루틴을 사용하여 개선해보자. 이전 코드와의 차이점은 함수 앞에 suspend 키워드가 붙어있다는 점 뿐이다.
```.kt
suspend fun login(credentials: Credentials): UserID
suspend fun loadUserData(userID: UserID): UserData
fun showData(data: UserData)

suspend fun showUserInfo(credentials: Credentials) {
  val userID = login(credentials)
  val userData = loadUserData(userID)
  showData(userData)
}
```
- 함수에 suspend 키워드가 붙은것은 어떤 의미일까?
- 이 변경자는 함수가 실행을 잠시 멈출 수도 있다는 뜻이다
- 예를 들어 네트워크 응답을 기다리는 경우 실행을 일시 중단할 수 있다
- 일시 중단은 기저 스레드를 블록시키지 않는다.
- 대신 함수 실행이 일시 중단되면 다른 코드가 같은 스레드에서 실행될 수 있다
![image](https://github.com/user-attachments/assets/35059f0d-6363-45bd-9b19-341b70d4d208)
- 이처럼 코드 구조를 변경하지 않았으며 코드는 여전히 순차적으로 보이고 동작하지만 블로킹 코드의 단점은 사라졌다
- 네트워크 응답을 기다리는 동안 기저 스레드는 다른 작업을 자유롭게 수행할 수 있다
- 물론 이 모든것이 마법처럼 동작하는 것은 아니며 login, loginUserData의 구현도 코틀린 코루틴을 고려해 작성되어야 한다
- 실제로 코틀린 생태계의 많은 라이브러리가 코루틴과 함께 작동하는 API를 제공한다
- 네트워크 요청의 경우 Ktor, Retrofit, OkHttp와 같은 라이브러리들이 코루틴을 고려해 작성되었다

## 5. 코루틴을 다른 접근 방법과 비교
- 코루틴이 어떤점이 다르고 더 나은지 콜백, 반응형스트림(RxJava), Future와 비교하여 살펴보자
- 이런 접근방법을 사용해본 적이 없다면 이부분은 넘어가도 좋다
```.kt
/* 콜백 사용 */
fun login(credentials: Credentials): UserID
fun loadUserData(userID: UserID): UserData
fun showData(data: UserData)

fun showUserInfo(credentials: Credentials) {
  loginAsync(credentails) { userID ->
    loadUserDataAsync(userID) {
      showData(userData)
    }
  }
}

/* Future 사용 */
/* 콜백 사용 */
fun login(credentials: Credentials): UserID
fun loadUserData(userID: UserID): UserData
fun showData(data: UserData)

fun showUserInfo(credentials: Credentials) {
  loginAsync(credentails)
    .thenCompose { loadUserDataAsync(it) }
    .thenAccept { showData(it) }
}

/* RxJava 사용 */
fun login(credentials: Credentials): UserID
fun loadUserData(userID: UserID): UserData
fun showData(data: UserData)

fun showUserInfo(credentials: Credentials) {
  login(credentials)
    .flatMap { loadUserData(it) }
    .doOnSuccess { showData(it) }
    .subscribe()
}
```
- 이처럼 다른 방식을 사용할땐 새로운 연산자가 필요하며 콜백이 반복되는 등 부가 비용이 존재한다
- 이와 비교하면 코루틴은 함수에 suspend 키워드만 추가하면 되므로 매우 간편하며 코드의 형태를 유지하며 blocking을 방지할 수 있다
- 16, 17 장에서 코루틴용 반응형 스트림인 Flow에 대해 자세히 다룬다
 
### 5-1 suspend 함수 호출
- 일시 중단 함수는 실행을 일시 중단할 수 있기 때문에 일반 코드 아무곳에서나 호출할 수는 없다
- suspend 함수는 suspend 함수 내에서만 호출할 수 있으며 또는 CoroutineScope(코루틴빌더)를 사용하여 코루틴을 실행해야 한다
- 물론 suspend 함수 내에서는 일반 함수는 호출할 수 있다 ([함수의 색 문제](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)를 이해하면 좋다)
- 만약 suspend 함수가 아닌 함수에서 suspend 함수를 호출하려고 하면 컴파일 에러가 발생한다
```.kt
suspend fun mySuspendingFunction() {}

fun main() {
  mySuspendingFunction()
}
```
- 더 범용적이고 강력한 방법은 코루틴 빌더를 사용하는것이다.

- ## 6. 코루틴의 세로 들어가기: 코루틴 빌더
- 코루틴을 생성할때는 코루틴 빌더 함수를 사용한다.
- runBlocking: 블로킹 코드와 일시 중단 함수의 세계를 연결할 때 쓰인다
- launch: 값을 반환하지 않는 새로운 코루틴을 시작할 때 쓰인다
- async: 비동기적으로 값을 계산할 때 쓰인다

### 6-1. runBlocking 함수
- 일반 블로킹 코드를 일시 중단의 세계로 연결하려면 runBlocking을 사용한다
- runBlocking은 새 코루틴을 생성하고 실행하며 해당 코루틴이 완료될 때 까지 현재 스레드를 블록시킨다
- 전달된 코드 블록 내에서는 일시 중단 함수를 호출할 수 있다

```.kt
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds

suspend fun doSomethingSlowly() {
    delay(500.milliseconds) // 500밀리초 일시 중단
    println("I'm done")
}

fun main() = runBlocking {
    doSomethingSlowly()
}
```
- 코루틴을 사용하는 이유가 스레드를 블록시키지 않기 위함인데 runBlocking은 왜 사용할까?
- runBlocking이 끝나는 시점까지 해당 스레드가 블록되지만, 그 내부에선 추가적인 자식 코루틴을 얼마든지 시작할 수 있다
- 이 자식 코루틴들은 다른 스레드를 블록시키지 않는다
- 대신 일시 중단될 때마다 하나의 스레드가 해방돼 다른 코루틴이 코드를 실행할 수 있게 된다

### 6-2. launch
- launch 함수는 새로운 자식 코루틴을 시작하는 데 쓰인다
- 이는 일반적으로 발사 후 망각(???) 시나리오에 사용되며 어떤 코드를 실행하되 그 결과값을 기다리지 않는 경우에 적합하다
- runBlocking이 오직 하나의 스레드만을 블로킹한다는 주장을 테스트해보자
```.kt
private var zeroTime = System.currentTimeMillis()
fun log(message: Any?) = 
    println("${System.currentTimeMillis() - zeroTime}" + 
    "[${Thread.currentThread().name}] $message)

fun main() = runBlocking {
   log("The first, parent, coroutine starts")
   launch {
       log("The second coroutine starts and is ready to be suspended")
       delay(100.milliseconds)
       log("The second coroutine is resumed")
   }
   launch {
       log("The third coroutine can run in the meantime")
   }
   log("The first coroutine has launched two more coroutines")
}
```
- 이 예제를 -Dkotlinx.coroutines.debug JVM 옵션과 함께 실행하거나 코틀린 플레이그라운드에서 실행하면 스레드 이름 옆에 코루틴 이름에 대한 추가 정보를 얻을 수 있고 이 정보는 코루틴이 어떻게 동작 하는지 이해하는데 큰 도움이 된다
![image](https://github.com/user-attachments/assets/9513ed20-21dc-4539-80e3-9892696c06dd)

```.kt
36 [main @coroutine#1] The first, parent, coroutine starts
40 [main @coroutine#1] The first coroutine has launched two more coroutines
42 [main @coroutine#2] The second coroutine starts and is ready to be suspended
47 [main @coroutine#3] The third coroutine can run in the meantime
149 [main @coroutine#2] The second coroutine is resumed
```
- 이 예시에서 모든 코루틴은 한 스레드 즉 main 스레드에서 실행된다
![image](https://github.com/user-attachments/assets/72a402de-b7f3-4461-8380-da8ab755a447)

- 이 코드에서는 3개의 코루틴이 시작된다
- 첫 번째는 runBlocking에 의해시작된 부모 코루틴(coroutine#1)이고 두 번째와 세 번째는 2번의 launch 호출에 의한 자식 코루틴이다
- coroutine#2가 delay를 호출하면 코루틴이 일시 중단되고, 메인 스레드는 다른 스레드는 다른 코루틴이 실행될 수 있게 해방된다. 이를 suspension point(일시 중단 지점)이라고 한다
- 이제 corouitne#2는 지정된 시간 동안 일시 중단되고 메인 스레드는 다른 코루틴이 실행될 수 있게 해방된다
- 그 결과 coroutine#3이 이 작업을 시작할 수 있다
- coroutine#3은 로그 호출 하나만 포함하고 있기 때문에 빠르게 끝난다. 지정된 100밀리초 후 coroutine#2가 작업을 재개하고 프로그램 전체가 완료된다
### 일시 중단된 코루틴은 어디로 가는가?
- 코루틴이 제대로 작동할 수 있게 하는 주요 작업은 컴파일러가 수행한다. 컴파일러는 코루틴을 일시 중단하고 재개하며 스케줄링하는 데 필요한 지원 코드를 생성한다.
- 일시 중단 함수의 코드는 컴파일 시점에 변환되고 실행 시점에 코루틴이 일시 중단될 때 해당 시점의 상태 정보가 메모리에 저장된다

###

- 동시성과 병렬성의 비교를 떠올려보면 지금은 단일 스레드에서 병렬성 없이 교차 실행되는 경우라는 것을 알 수 있다
- 여러 스레드에서 병렬 실행하는 경우 다중 스레드 디스패처를 사용할 수 있다(14.7에서 다룸)
- launch는 Job타입의 객체를 반환하는데 이를 시작된 코루틴에 대한 핸들로 생각할 수 있다
- launch는 시작 후 신경쓰지 않아도 되는 작업에 적합한 코루틴 빌더 이다

### 6-3. 대기 가능한 연산: async 빌더
- async는 launch와 달리 Deferred<T> 인스턴스를 반환한다
- Deferred를 사용해 주로 할 일은 await이라는 일시 중단 함수로 그 결과를 기다리는 것이다
- 예를들어 두 숫자를 비동기적으로 계산하는 예제를 보자
- delay호출을 통해 계산을 오래 걸리는 것 처럼 시뮬레이션 한다
```.kt
suspend fun slowlyAddNumbers(a: Int, b: Int): Int {
    log("Waiting a bit before calculating $a + $b")
    delay(100.milliseconds * a)
    return a + b
}
 
 
fun main() = runBlocking {
    log("Starting the async computation")
    val myFirstDeferred = async { slowlyAddNumbers(2, 2) }
    val mySecondDeferred = async { slowlyAddNumbers(4, 4) }
    log("Waiting for the deferred value to be available")
    log("The first result: ${myFirstDeferred.await()}")
    log("The second result: ${mySecondDeferred.await()}")
}
```
- 결과
```.kt
0 [main @coroutine#1] Starting the async computation
4 [main @coroutine#1] Waiting for the deferred value to be available
8 [main @coroutine#2] Waiting a bit before calculating 2 + 2
9 [main @coroutine#3] Waiting a bit before calculating 4 + 4
213 [main @coroutine#1] The first result: 4
415 [main @coroutine#1] The second result: 8
```
- 타임 스탬프를 보면 두 값을 계산하는데 총 약 400밀리초가 걸렸음을 알 수 있다
- async를 사용하여 두 계산이 병렬적으로 일어나도록 했다
- async의 결과의 Deffered에서 await을 호출하면 그 결과값이 사용 가능해질때까지 코루틴이 일시 중단된다
![image](https://github.com/user-attachments/assets/cd437c1e-cf9c-4601-8655-66eeaf84e627)

- Deffered는 미래에 언젠가는 값을 알게 될 것이라는 약속, 연기된 값을 나타낸다
- 기본적인 코드에서 일시 중단 함수를 순차적으로 호출할 때는 async와 await을 사용할 필요가 없다
- 여러 독립적인 작업을 동시에 실행하고 그 결과를 기다릴때만 async를 사용하면 된다
### async await은 키워드가 아닌 함수
- 다른 언어와 달리 async, await은 키워드가 아닌 코틀린이 제공하는 메서드이다

## 7. 어디서 코드를 실행할지 정하기: 디스패처
- 코루틴의 디스패처는 코루틴을 실행할 스레드를 결정한다
- 디스패처를 선택함으로써 코루틴을 특정 스레드로 제한 또는 스레드 풀에 분산시킬 수 있다

### 7-1. 디스패처 선택
- 코루틴은 기본적으로 부모 코루틴에서 디스패처를 상속 받으므로 모든 코루틴에 대해 명시적으로 디스패처를 지정할 필요는 없다
- 아래는 직접 디스패처를 선택할 때 사용할 수 있는 디스패처 종류를 소개한다

#### 다중 스레드를 사용하는 범용 디스패처: Dispatchers.Default
- 가장 일반적인 디스패처
- CPU 코어 수만큼의 스레드로 구성된 스레드 풀을 기반으로 한다
- 기본 디스패처에서 코루틴을 스케줄링하면 여러 스레드에서 코루틴이 분산돼 실행되며 멀티코어 시스템에서는 병렬로 실행될 수 있다
- 특별한 제약이 없는 경우 기본 디스패처를 사용하는 것이 적합하다

#### Dispatchers.Main
- UI 프레임워크 (JavaFX, AWT, Swing, Android)등을 사용할 때 특정 작업을 UI 스레드(메인 스레드)라고 불리는 특정 스레드에서 실행해야 할 때 사용한다
- 애플리케이션의 UI나 메인 스레드가 무엇인지에 대한 보편적인 정의가 없기 때문에 Dispatchers.Main의 실제 값은 사용하는 프레임워크에 따라 다르다
- org.jetbrains.kotlinx:kotlinx-coroutines-swing이나 org.jetbrains.kotlinx:kotlinx-coroutines-javafx 같은 추가 아티팩트는 각각 자신의 프레임워크에 따른 메인 디스패처 구현을 제공한다

##### 블로킹되는 IO 작업 처리: Dispatchers.IO
- 서드파티 라이브러리를 사용할 때 코루틴을 염두에 두고 설계된 API를 선택할 수 없는 경우가 있다
- 데이터베이스, 네트워크 등의 IO 작업을 처리할 때 Dispatchers.Default에서 실행하게 되면 스레드풀이 소진돼 다른 코루틴은 완료될 때 까지 실행되지 못한다
- Dispatchers.IO는 이러한 상황을 처리하기 위해 설계됐다. 이 디스패처에서 실행된 코루틴은 자동으로 확장되는 스레드 풀에서 실행되며 CPU 집약적이지 않은 작업에 적합하다

|디스패처|스레드 개수|쓰임새|
|------|---|---|
|Dispatchers.Default|CPU 코어 수|일반적인 연산, CPU 집약적인 작업|
|Dispatchers.Main|1|UI 프레임워크의 맥락에서만 UI 작업|
|Dispatchers.IO|64 + CPU 코어 개수(단 최대 64개만 병렬 실행)|블로킹 IO, 네트워크 작업, 파일 작업|
|Dispatchers.Unconfined|아무 스레드나|즉시 스케줄링해야 하는 특별한 경우|
|limitedParallelism(n)|커스텀(n)|커스텀 시나리오|

![image](https://github.com/user-attachments/assets/a3197e12-9a6e-4c44-8595-851a5c712abf)

### 7-2. 코루틴 빌더에 디스패처 전달
```.kt
fun main() {
    runBlocking {
        log("Doing some work")
        launch(Dispatchers.Default) {
            log("Doing some background work")
        }
    }
}

26 [main @coroutine#1] Doing some work
33 [DefaultDispatcher-worker-1 @coroutine#2] Doing some background work
```
- coroutine#2가 Default 에서 실행된 것을 볼 수 있다

### 7-3. withContext를 사용해 코루틴 안에서 디스패처 바꾸기
```.kt
launch(Dispatchers.Default) {
    val result = performBackgroundOperation()
    withContext(Dispatchers.Main) {
        updateUI(result)
    }
}
```
![image](https://github.com/user-attachments/assets/55abdff7-351f-4b0c-8a92-de06fbb61987)

### 7-4 코루틴과 디스패처는 스레드 안정성 문제에 대한 마법 같은 해결책이 아니다
- Default, IO 등의 디스패처는 멀티 스레드를 사용한다
- 단일 코루틴에서는 문제가 되지 않을 수 있지만 여러 코루틴을 실행하는 경우 동기화 문제를 겪을 수 있다

```.kt
fun main() {
    runBlocking {
        launch(Dispatchers.Default) {
            var x = 0
            repeat(10_000) {
                x++
            }
            println(x)
        }
    }
}
 
// 10,000
```
- 위 예제에서 단일 코드의 경우 예상한 결과값 10000이 출력된다

```.kt
fun main() {
    runBlocking {
        var x = 0
        repeat(10_000) {
            launch(Dispatchers.Default) {
                x++
            }
        }
        delay(1.seconds)
        println(x)
    }
}
// 9,916
```
- 여러 코루틴이 같은 데이터를 수정하고 있기 때문에 일부 증가 작업이 서로의 결과를 덮어쓰는 상황이 발생할 수 있다
- 이런 경우 코루틴에서는 Mutex를 사용한다
```.kt
fun main() = runBlocking {
   val mutex = Mutex()
   var x = 0
   repeat(10_000) {
       launch(Dispatchers.Default) {
           mutex.withLock {
               x++
           }
       }
   }
   delay(1.seconds)
   println(x)
}
// 10000
```
- 또한 AtomicInterger나 ConcurrentHashMap 같은 병렬 변경을 위해 설계된 원자적이고 스레드 안전한 구조를 사용할 수 있다
## 8. 코루틴은 코루틴 context에 추가적인 정보를 담고 있다
- 우리는 아까 withContext에 Dispatcher를 인자로 전달했다.
- 하지만 이 함수의 파라미터 타입은 CoroutineDispatcher가 아닌 CoroutineContext다
- CoroutineContext는 코루틴의 추가적인 문맥 정보를 담고 있다
- CoroutineContext를 확인하려면 어떤 일시 중단 함수 안에서든 coroutineContext라는 특별한 속성에 접근하면 된다
- 이것은 코틀린 컴파일러의 고유 기능이며 실제 구현은 코틀린 컴파일러에 의해 특별히 처리된다
```.kt
import kotlin.coroutines.coroutineContext
 
suspend fun introspect() {
    log(coroutineContext)
}
 
fun main() {
    runBlocking {
        introspect()
    }
}
 
// 25 [main @coroutine#1] [CoroutineId(1),
    "coroutine#1":BlockingCoroutine{Active}@610694f1,
    BlockingEventLoop@43814d18]
```
- 코루틴 빌더나 withContext 함수에 인자를 전달하면 자식 코루틴의 context에서 해당 요소를 덮어쓴다
- 예를 들어 runBlocking 코루틴을 IO 디스패처에서 실행하면서 이름을 "Coroutine"으로 설정할 수 있다
```.kt
fun main() {
   runBlocking(Dispatchers.IO + CoroutineName("Coolroutine")) {
       introspect()
   }
}

// 27 [DefaultDispatcher-worker-1 @Coolroutine#1]
  [CoroutineName(Coolroutine), CoroutineId(1),
   "Coolroutine#1":BlockingCoroutine{Active}@d115c9f, Dispatchers.IO]
```
![image](https://github.com/user-attachments/assets/65d451dd-238c-4949-81da-ecd6ddcb1bf0)
