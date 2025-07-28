## 0.Objectives
- 오류, 예외 발생 시 코드의 동작 제어
- 오류 처리를 구조적 동시성 개념과 연결하는 방법
- 시스템 일부가 실패해도 정상적으로 작동하는 코드를 작성하는 방법
- 동시성 코드를 위한 단위 테스트 작성법
- 테스트 실행 속도를 높이고 세밀한 동시성 제약 조건을 테스트 하는 방법
- 터빈 라이브러리를 사용한 플로우 테스트

## 1. 코루틴 내부에서 던져진 오류 처리
- 코루틴에서 launch나 async 호출을 try-catch로 감싸는 것은 의미가 없다
- 코루틴 빌더는 실행할 새로운 코루틴을 생성하는데, 이 새로운 코루틴에서 발생한 예외는 catch 블록에 잡히지 않는다
```.kt
import kotlinx.coroutines.*
 
fun main(): Unit = runBlocking {
    try {
        launch {
            throw UnsupportedOperationException("Ouch!")
        }
    } catch (u: UnsupportedOperationException) {
        println("Handled $u")
    }
}
// Exception in thread "main" java.lang.UnsupportedOperationException: Ouch!
//  at MyExampleKt$main$1$1.invokeSuspend(MyExample.kt:6)
//       ...
```

- 이 예외를 올바르게 처리하는 한 가지 방법은 launch에 전달되는 람다 블록 안에 try-catch 블록을 넣는 것이다

```.kt
import kotlinx.coroutines.*

fun main(): Unit = runBlocking {
   launch {
       try {
           throw UnsupportedOperationException("Ouch!")
       } catch (u: UnsupportedOperationException) {
           println("Handled $u")
       }
   }
}
// Handled java.lang.UnsupportedOperationException: Ouch!
```
- async로 생성된 코루틴이 예외를 던진다면 그 결과에 대해 await을 호출할 때 이 예외가 다시 발생한다
- 따라서 아래처럼 처리해야 한다
```.kt
import kotlinx.coroutines.*

fun main(): Unit = runBlocking {
   val myDeferredInt: Deferred<Int> = async {
       throw UnsupportedOperationException("Ouch!")
   }
   try {
       val i: Int = myDeferredInt.await()
       println(i)
   } catch (u: UnsupportedOperationException) {
       println("Handled: $u")
   }
}
```
- 하지만 동시에 오류 콘솔에도 예외가 출력된다
- 이는 await가 예외를 다시 던지지만 원래의 예외도 여전히 관찰되기 때문이다
- 여기서는 async가 예외를 부모 코루틴인 runBlocking에 전파하고 프로그램은 종료된다

## 2. 코틀린 코루틴에서의 오류 전파
- 두 가지 경우를 나누어 살펴본다
- 첫번째로 작업을 동시적으로 분해해 처리하는 경우 자식의 실패는 더 이상 최종 결과를 얻을 수 없다는 것을 의미한다
- 이 경우 부모 코루틴도 예외로 완료돼야 하며 한 자식의 실패가 부모의 실패로 이어진다
- 두번째로 자식의 실패가 전체의 실패로 이어지지 않기를 원할때이다
- 이 두가지 상황을 나누어 Job vs SupervisorJob을 살펴본다

### 2.1 자식이 실패하면 모든 자식을 취소하는 코루틴
- 코루틴 콘텍스트를 설명할 때 코루틴 간의 부모-자식 계층이 Job 객체를 통해 구축된다는 점을 배웠다
- 실패한 자식 코루틴은 자신의 실패를 부모에게 전파한다
- 그러면 부모는 다음을 수행한다
    - 불필요한 작업을 막기 위해 다른 모든 자식을 취소
    - 같은 예외를 발생시키면서 자신의 실행을 완료
    - 자신의 상위 계층으로 예외 전파
<img width="500" height="199" alt="image" src="https://github.com/user-attachments/assets/1392e731-4168-4533-8baf-c1347c605015" />

- 취소를 관리할 필요가 없는 이것은 코루틴의 큰 장점이다
- 아래 코드에서 2가지 하트비트 역할을 하는 코루틴과 1초뒤 예외를 수행하는 2가지 코루틴을 만든다
```.kt
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds
import kotlin.time.Duration.Companion.seconds

fun main(): Unit = runBlocking {
   launch {
       try {
           while (true) {
               println("Heartbeat!")
               delay(500.milliseconds)
           }
       } catch (e: Exception) {
           println("Heartbeat terminated: $e")
           throw e
       }
   }
   launch {
       delay(1.seconds)
       throw UnsupportedOperationException("Ow!")
   }
}
```
```
Heartbeat!
Heartbeat!
Heartbeat terminated: kotlinx.coroutines.JobCancellationException: Parent job is Cancelling; job=BlockingCoroutine{Cancelling}@1517365b
Exception in thread "main" java.lang.UnsupportedOperationException: Ow!
```
- 이러한 오류 전파 동작은 launch 뿐 아니라 async로 시작해도 같은 동작을 볼 수 있다

### 2.2 구조적 동시성은 코루틴 스코프를 넘는 예외에만 영향을 미친다
```.kt
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds
import kotlin.time.Duration.Companion.seconds

fun main(): Unit = runBlocking {
   launch {
       try {
           while (true) {
               println("Heartbeat!")
               delay(500.milliseconds)
           }
       } catch (e: Exception) {
           println("Heartbeat terminated: $e")
           throw e
       }
   }
   launch {
       try {
           delay(1.seconds)
           throw UnsupportedOperationException("Ow!")
       } catch(u: UnsupportedOperationException) {
           println("Caught $u")
       }
   }
}
```
```
Heartbeat!
Heartbeat!
Caught java.lang.UnsupportedOperationException: Ow!
Heartbeat!
Heartbeat!
...
```
- 이렇게 각 코루틴 내에서 예외를 잡으면 예외가 전파되지 않는다
- 다만 예외를 잡을때는 앞에서 배운 것 처럼 CancellationException과 그 하위 타입의 예외에 주의해야 한다
- 취소는 코루틴 생명주기의 자연스러운 부분이기 때문에 이런 예외를 코드가 삼켜서는 안 된다

### 2.3 Supervisor는 부모와 형제가 취소되지 않게 한다
- 슈퍼바이저는 자식이 실패하더라도 생존한다
- 다른 자식 코루틴을 취소하지 않으며 예외를 구조적 동시성 상위로 전파하지 않는다
<img width="481" height="158" alt="image" src="https://github.com/user-attachments/assets/25dddddb-25f4-412e-9bb4-a3851f4afeaa" />

- 코루틴이 슈퍼바이저가 되려면 그 코루틴에 연관된 Job이 일반적인 Job이 아니라 SupervisorJob이어야 한다
- 슈퍼바이저의 동작을 직접 확인하려면 supervisorScope 함수를 사용해 스코프를 만들 수 있다
- coroutineScope 함수와 비슷하지만 자식 중 하나가 실패해도 형제 코루틴이 종료되지 않고 처리되지 않은 예외는 더 이상 전파되지 않는다
```.kt
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds
import kotlin.time.Duration.Companion.seconds

fun main(): Unit = runBlocking {
   supervisorScope {
       launch {
           try {
               while (true) {
                   println("Heartbeat!")
                   delay(500.milliseconds)
               }
           } catch (e: Exception) {
               println("Heartbeat terminated: $e")
               throw e
           }
       }
       launch {
           delay(1.seconds)
           throw UnsupportedOperationException("Ow!")
       }
   }
}
```
- 이 코드의 결과를 보면 예외가 발생한 후에도 하트비트 코루틴이 계속 작동하는 것을 확인할 수 있다
```
Heartbeat!
Heartbeat!
Exception in thread "main" java.lang.UnsupportedOperationException: Ow!
...
Heartbeat!
Heartbeat!
...
```
- 이런 동작이 가능한 이유는 SupervisorJob이 launch 빌더로 시작된 자식 코루틴에 대해 CoroutineExceptionHandler를 호출하기 때문이다

## 3. CoroutineExceptionHandler: 예외 처리를 위한 마지막 수단
- 에러가 전파되면서 부모 코루틴까지 도달했지만 처리되지 않는 예외는 CoroutineExceptionHandler라는 특별한 핸들러에게 전달된다.
- 이 핸들러는 CoroutineContext의 일부이며 예외 핸들러가 없다면 처리되지 않는 예외는 시스템 전역 예외 핸들러로 이동한다
- CoroutineExceptionHandler를 코루틴 콘텍스트에 제공하면 처리되지 않는 예외를 처리하는 동작을 정의할 수 있다
```.kt
val exceptionHandler = CoroutineExceptionHandler { context, exception -> println("[ERROR] $exception")}
```
```.kt
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.seconds

class ComponentWithScope(dispatcher: CoroutineDispatcher = Dispatchers.Default) {
   private val exceptionHandler = CoroutineExceptionHandler { _, e ->
      println("[ERROR] ${e.message}")
   }

   private val scope = CoroutineScope(
       SupervisorJob() + dispatcher + exceptionHandler
   )

   fun action() = scope.launch {
      throw UnsupportedOperationException("Ouch!")
   }
}


fun main() = runBlocking {
   val supervisor = ComponentWithScope()
   supervisor.action()
   delay(1.seconds)
}
```
- 코루틴은 처리되지 않는 예외의 처리를 부모에게 위임한다
- 이 작업이 반복된다. 따라서 중간에 있는 CoroutineExceptionHandler라는 것은 존재하지 않는다
- 루트 코루틴이 아닌 코루틴 콘텍스트에 설치된 핸들러는 결코 사용되지 않는다
- 중간에 있는 핸들러는 쓰이지 않는다는 사실을 확인하려면 아래 코드를 확인해보자
```.kt
import kotlinx.coroutines.*
 
private val topLevelHandler = CoroutineExceptionHandler { _, e ->
    println("[TOP] ${e.message}")
}
 
private val intermediateHandler = CoroutineExceptionHandler { _, e ->
    println("[INTERMEDIATE] ${e.message}")
}
 
@OptIn(DelicateCoroutinesApi::class)
fun main() {
    GlobalScope.launch(topLevelHandler) {
        launch(intermediateHandler) {
            throw UnsupportedOperationException("Ouch!")
        }
    }
    Thread.sleep(1000)
}
// [TOP] Ouch!
```
- 이는 예외가 여전히 부모 코루틴에게 전파될 수 있기 때문이다
- launch 호출에 예외 핸들러가 있음에도 루트 코루틴이 아니기 때문에 예외가 계속해서 코루틴 계층을 따라 전파된다
<img width="1021" height="641" alt="image" src="https://github.com/user-attachments/assets/6ade8b22-9668-463e-bbe7-8acf2f50be59" />

### 3.1 CoroutineExceptionHandler를 launch와 async에 적용할 때의 차이점
- CoroutineExceptionHandler를 살펴볼 때 예외 핸들러는 계층의 최상위 코루틴이 launch로 생성된 경우에만 호출된다
```.kt
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.seconds
 
class ComponentWithScope(dispatcher: CoroutineDispatcher = 
 Dispatchers.Default) {
    private val exceptionHandler = CoroutineExceptionHandler { _, e ->
        println("[ERROR] ${e.message}")
    }
 
 
    private val scope = CoroutineScope(SupervisorJob() + dispatcher + 
     exceptionHandler)
 
 
    fun action() = scope.launch {
        async {
            throw UnsupportedOperationException("Ouch!")
        }
    }
}
 
 
fun main() = runBlocking {
    val supervisor = ComponentWithScope()
    supervisor.action()
    delay(1.seconds)
}
// [Error] Ouch!
```
- 여기서 최상위 코루틴을 async로 시작하도록 구현을 변경하면 코루틴 예외 핸들러가 호출되지 않는 것을 확인할 수 있다
```.kt
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.seconds
 
class ComponentWithScope(dispatcher: CoroutineDispatcher = 
 Dispatchers.Default) {
    private val exceptionHandler = CoroutineExceptionHandler { _, e ->
        println("[ERROR] ${e.message}")
    }
 
 
    private val scope = CoroutineScope(SupervisorJob() + dispatcher + 
     exceptionHandler)
 
 
    fun action() = scope.async {
        launch {
            throw UnsupportedOperationException("Ouch!")
        }
    }
}
 
 
fun main() = runBlocking {
    val supervisor = ComponentWithScope()
    supervisor.action()
    delay(1.seconds)
}
// No output is printed
```
- 이 경우에는 예외에 대한 책임이 await()을 호출하는 Deferred 소비자에게 있다
- 따라서 await 호출을 try-catch로 감싸는 방식으로 예외를 처리할 수 있다
- 이 예제의 scope에 SupervisorJob이 없었다면 처리되지 않은 예외가 다른 자식 코루틴을 모두 취소했을 것이다

## 4. 플로우에서 예외처리
- 예외 발생 예시
```.kt
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class UnhappyFlowException: Exception()

val exceptionalFlow = flow {
   repeat(5) { number ->
       emit(number)
   }
   throw UnhappyFlowException()
}
```
- 일반적으로 플로우의 일부분에서 예외가 발생하면 collect에서 예외가 던져진다
- collect 호출을 try-catch로 감싸면 예상대로 동작한다는 뜻이다
- 이때 플로우에 중간 연산자가 적용됐는지 여부와는 관계가 없다
```.kt
fun main() = runBlocking {
   val transformedFlow = exceptionalFlow.map {
       it * 2
   }
   try {
       transformedFlow.collect {
           print("$it ")
       }
   } catch (u: UnhappyFlowException) {
       println("\nHandled: $u")
   }
   // 0 2 4 6 8
   // Handled: UnhappyFlowException
}
```

### 4.1 catch 연산자로 업스트림 예외 처리
- catch 연산자는 업스트림의 예외를 잡아낸다
```.kt
fun main() = runBlocking {
   exceptionalFlow
       .catch { cause ->
           println("\nHandled: $cause")
           emit(-1)
       }
       .collect {
           print("$it ")
       }
}
// 0 1 2 3 4
// Handled: UnhappyFlowException
// -1
```
- 다운 스트림의 예외는 잡아내지 않는다
```.kt
fun main() = runBlocking {
   exceptionalFlow
       .map {
           it + 1
       }
       .catch { cause ->
           println("\nHandled $cause")
       }
       .onEach {
           throw UnhappyFlowException()
       }
       .collect()
}
// Exception in thread "main" UnhappyFlowException
```
<img width="601" height="157" alt="image" src="https://github.com/user-attachments/assets/e2f174b8-5231-4ba7-ae35-b39031a64a93" />

### 4.2 술어가 참일 때 플로우의 수집 재시도: retry 연산자
- 예외가 발생했을 때 단순히 오류 메시지와 함께 종료하는 대신 작업을 재시도하고 싶다면
- retry 연산자를 사용할 수 있다
- catch와 마찬가지로 업스트림의 예외를 잡으며 
- 예외를 처리하고 Boolean값을 반환하는 람다를 사용할 수 있다 / true반환시 재시도
```.kt
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.*
import kotlin.random.Random

class CommunicationException : Exception("Communication failed!")

val unreliableFlow = flow {
   println("Starting the flow!")
   repeat(10) { number ->
       if (Random.nextDouble() < 0.1) throw CommunicationException()
       emit(number)
   }
}

fun main() = runBlocking {
   unreliableFlow
       .retry(5) { cause ->
           println("\nHandled: $cause")
           cause is CommunicationException
       }
       .collect { number ->
           print("$number ")
       }
}
```
- 난수이기 때문에 출력이 다를 수 있지만 아래와 같은 비슷한 출력을 볼 수 있다
```
Starting the flow!
0 1 2 3 4
Handled: CommunicationException: Communication failed!
Starting the flow!
0 1 2 3
Handled: CommunicationException: Communication failed!
Starting the flow!
0 1 2 3 4 5 6 7 8 9
```
- 재시도할 때는 업스트림 연산자가 모두 다시 실행된다는 점을 기억해야 한다

## 5. 코루틴과 플로우 테스트
- 테스트 환경에서 runBlocking을 사용하는 경우 테스트에 delay와 같은 시간 지연이 모두 수행되어 테스트의 속도를 저하시킬 수 있다
- 이런 상황을 위해 코루틴 테스트에서 runTest를 제공한다

### 5.1 코루틴을 사용하는 테스트를 빠르게 만들기: 가상 시간과 테스트 디스패처
- 테스트를 실시간으로 실행해서 지연을 기다리는 대신 코틀린 코루틴은 가상 시간을 사용해 테스트를 빠르게 실행할 수 있다
-  아래 에제는 runTest를 사용해 가상 시간으로 테스트를 실행한다
- 20초의 delay가 있음에도 몇 밀리초만에 테스트가 수행된다
```.kt
import kotlinx.coroutines.*
import kotlinx.coroutines.test.*
import kotlin.test.*
import kotlin.time.Duration.Companion.seconds

class PlaygroundTest {
   @Test
   fun testDelay() = runTest {
       val startTime = System.currentTimeMillis()
       delay(20.seconds)
       println(System.currentTimeMillis() - startTime)
  // 11
   }
}
```
- runTest의 디스패처는 단일 쓰레드이며 테스트를 위한 특별한 디스패처를 사용한다
- runBlocking과 마찬가지로 runTest를 사용할 때도 launch로 시작한 코루틴이 단언문이 실행되기 전에 실행되게 할 수 있는 방법이 없기때문에
- 아래 단언문은 실패한다
```.kt
@Test
fun testDelay() = runTest {
   var x = 0
   launch {
       x++
   }
   launch {
       x++
   }
   assertEquals(2, x)
}
```
- 만약 코드에 yield, delay등 일시 중단 함수를  추가하면 이 테스트가 통과한다.
- 테스트 디스패처에서는 TestCoroutineScheduler를 통해 가상 시간을 더 세밀하게 제어할 수 있다
- runTest는 TestScope라는 특수한 스코프에 접근할 수 있으며 TestCoroutineScheduler의 기능을 사용할 수 있게 해준다
- runCurrent: 현재 실행하게 예약된 모든 코루틴을 실행한다
- advancedUntilIdle: 예약된 모든 코루틴을 실행한다
- 가상 디스패처의 시계를 단순히 앞으로 이동시키려면 delay를 사용할 수 있다
- 가상 디스패처의 시간이 궁금하면 currentTime속성을 사용할 수도 있다
```.kt
@OptIn(ExperimentalCoroutinesApi::class)
@Test
fun testDelay() = runTest {
   var x = 0
   launch {
       delay(500.milliseconds)
       x++
   }
   launch {
       delay(1.second)
       x++
   }
   println(currentTime) // 0

   delay(600.milliseconds)
   assertEquals(1, x)
   println(currentTime) // 600

   delay(500.milliseconds)
   assertEquals(2, x)
   println(currentTime) // 1100
}
```
- 현재 가상 시간을 기준으로 스케줄되어있는 모든 코루틴을 실행할 때는 runCurrent함수를 사용할 수 있다
- 미래의 어느 시점에 예약된 코루틴까지 실행하려면 advancedUntilIdle 함수를 사용할 수 있다
```.kt
@OptIn(ExperimentalCoroutinesApi::class)
@Test
fun testDelay() = runTest {
   var x = 0
   launch {
       x++
       launch {
           x++
       }
   }
   launch {
       delay(200.milliseconds)
       x++
   }
   runCurrent()
   assertEquals(2, x)
   advanceUntilIdle()
   assertEquals(3, x)
}
```
- Dispatchers.Default같은 일반 디스패처는 테스트 환경에 대한 아무런 정보를 갖고 있지 않기 때문에 테스트 환경에 적합하지 않다
- 따라서 외부에서 Dispatcher를 주입할 수 있도록 구조를 설계하여 테스트 환경에서 테스트 디스패처를 사용할 수 있도록 dispatcher를 파라미터로 받는게 적절하다

### 5.2 터빈으로 플로우 테스트
```.kt
val myFlow = flow {
   emit(1)
   emit(2)
   emit(3)
}

@Test
fun doTest() = runTest {
   val results = myFlow.toList()
   assertEquals(3, results.size)
}
```
```.kt
@Test
fun doTest() = runTest {
   val results = myFlow.test {
       assertEquals(1, awaitItem())
       assertEquals(2, awaitItem())
       assertEquals(3, awaitItem())
       awaitComplete()
   }
}
```
