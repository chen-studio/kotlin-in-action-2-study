## 0.Objectives
- 코루틴을 관리하여 취소, 오류를 제대로 처리하자
- `Structured Concurrency`(구조화된 동시성)을 통해 각 애플리케이션 안에서 코루틴과 그 생애 주기의 계층을 관리하고 추적할 수 있는 기능이 코루틴에 내장되어 있다
- 구조화된 동시성 메커니즘을 자세히 살펴보고, 많은 수의 코루틴을 효과적으로 관리해보자

## 1. Coroutine Scope가 코루틴 간의 구조를 확립한다
- launch, async같은 코루틴 빌더 함수들은 사실 CoroutineScope 인터페이스의 확장함수다
- launch나 async를 사용해 새로운 코루틴을 만들면 이 새로운 코루틴은 자동으로 해당 코루틴의 자식이 된다

```.kt
fun main() {
    runBlocking { // this: CoroutineScope
        launch { // this: CoroutineScope
            delay(1.seconds)
            launch {
                delay(250.milliseconds)
                log("Grandchild done")
            }
            log("Child 1 done!")
        }
        launch {
            delay(500.milliseconds)
            log("Child 2 done!")
        }
        log("Parent done!")
    }
}
```
- 출력을 보면 runBlocking 함수 본문이 거의 즉시 실행을 마쳤음에도 모든 자식 코루틴이 완료될 때 까지 프로그램이 종료되지 않는다

```.kt
29 [main @coroutine#1] Parent done!
539 [main @coroutine#3] Child 2 done!
1039 [main @coroutine#2] Child 1 done!
1293 [main @coroutine#4] Grandchild done
```

- runBlocking은 여전히 어떤 자식 코루틴이 작업 중인지 알고, 모든 작업이 완료될 때 까지 기다린다
- 아래 그림을 보면 더 자세히 알 수 있다
- ![image](https://github.com/user-attachments/assets/27d1bdcc-56a8-41d2-ae8b-df784d48fca9)


### 1-1 코루틴 스코프 생성: coroutineScope 함수
- 코루틴 빌더로 새로운 코루틴을 만들면 자체적인 CoroutineScope를 생성한다
- 하지만 새 코루틴을 만들지 않고도 코루틴 스코프를 그룹화 할 수 있는데, coroutineScope 함수를 사용할 수 있다
- coroutineScope 함수의 전형적인 사용 사례는 `동시적 작업 분해(concurrent decomposition of work)`, 즉 여러 코루틴을 활용해 계산을 수행하는 것이다
- coroutineScope이 값을 반환할 수 있다는 사실을 이용해 두 값의 합계를 반환하고 이를 로그에 남긴다
```.kt
import kotlinx.coroutines.*
import kotlin.random.Random
import kotlin.time.Duration.Companion.milliseconds
 
suspend fun generateValue(): Int {
    delay(500.milliseconds)
    return Random.nextInt(0, 10)
}
 
suspend fun computeSum() {
    log("Computing a sum...")
    val sum = coroutineScope {
        val a = async { generateValue() }
        val b = async { generateValue() }
        a.await() + b.await()
    }
    log("Sum is $sum")
}
 
fun main() = runBlocking {
    computeSum()
}
 
// 0 [main @coroutine#1] Computing a sum...
// 532 [main @coroutine#1] Sum is 10
```
- ![image](https://github.com/user-attachments/assets/e831d019-9ea8-4d2c-8f03-289c391ae1a5)

### 1-2 코루틴 스코프를 컴포넌트와 연관시키기: CoroutineScope
- coroutineScope 함수가 작업을 분해하는 데 사용되는 반면, 구체적 생명주기를 정의하고 동시 처리나 코루틴의 시작과 종료를 관리하는 클래스를 만들고 싶을 때 사용
- coroutineScope와 달리 실행을 일시 중단 하지 않으며 새로운 코루틴을 시작할 때 쓸 수 있는 새로운 코루틴 스코프를 생성
- CoroutineScope는 CoroutineContext를 파라미터로 받는다
- CoroutineScope를 디스패처만으로 호출하면 새로운 Job이 자동으로 생성된다
- 대부분의 실무에서는 부모로 에러를 전파하지 않는 SupervisorJob을 사용하는 것이 좋다
- 아래는 코루틴의 시작과 취소 예시를 보여준다
```.kt
class ComponentWithScope(dispatcher: CoroutineDispatcher = 
 Dispatchers.Default) {
 
    private val scope = CoroutineScope(dispatcher + SupervisorJob())
 
    fun start() {
        log("Starting!")
        scope.launch {
            while(true) {
                delay(500.milliseconds)
                log("Component working!")
            }
        }
        scope.launch {
            log("Doing a one-off task...")
            delay(500.milliseconds)
            log("Task done!")
        }
    }
 
    fun stop() {
        log("Stopping!")
        scope.cancel()
    }
}
```

- Component 클래스의 인스턴스를 생성하고 start를 호출하면 컴포넌트 내부에서 코루틴이 시작된다
- stop을 호출하면 컴포넌트의 생명주기가 종료된다

```.kt
fun main() {
   val c = ComponentWithScope()
   c.start()
   Thread.sleep(2000)
   c.stop()
}
// 22 [main] Starting!
// 37 [DefaultDispatcher-worker-2 @coroutine#2] Doing a one-off task...
// 544 [DefaultDispatcher-worker-1 @coroutine#2] Task done!
// 544 [DefaultDispatcher-worker-2 @coroutine#1] Component working!
// 1050 [DefaultDispatcher-worker-1 @coroutine#1] Component working!
// 1555 [DefaultDispatcher-worker-1 @coroutine#1] Component working!
// 2039 [main] Stopping!
```

- 생명주기를 관리해야 하는 컴포넌트를 다루는 프레임워크에서는 내부적으로 CoroutineScope를 많이 사용한다

#### coroutineScope vs CoroutineScope
- coroutineScope는 suspend 함수로 suspend 함수 내에서만 호출할 수 있으며 작업을 동시성으로 실행하기 위해 분해할 때 사용한다
- 여러 코루틴을 시작하고 모두 완료될때 까지 기다리며 결과를 계산할 수 있다
- CoroutineScope는 코루틴을 클래스의 생명주기와 연관시키는 영역을 생성할 때 쓰인다
- 이 함수는 영역을 생성하지만 추가 작업을 기다리지 않고 즉시 반환되며 반환된 코루틴 스코프를 취소할 수 있다
- 실무에서는 coroutineScope가 CoroutineScope 생성자 함수보다 더 많이 사용된다고 함

### 1-3 GlobalScope의 위험성
- 일부 예제에서 GlobalScope를 사용한다. 이름에서 알 수 있듯이 전역 수준에 존재하는 코루틴 스코프다
- 코루틴을 처음 접하는 사람에겐 매력적으로 보일 수 있지만 몇 가지 단점이 존재한다
- GlobalScope를 사용하면 전역 범위에서 시작되기 때문에 코루틴은 자동으로 취소되지 않으며 생명주기에 대한 개념도 없다
- 따라서 리소스 누수가 발생하거나 불필요한 작업이 지속될 수 있다
```.kt
fun main() {
    runBlocking {
        GlobalScope.launch {
            delay(1000.milliseconds)
            launch {
                delay(250.milliseconds)
                log("Grandchild done")
            }
            log("Child 1 done!")
        }
        GlobalScope.launch {
            delay(500.milliseconds)
            log("Child 2 done!")
        }
        log("Parent done!")
    }
}
 
// 28 [main @coroutine#1] Parent done!
```
- ![image](https://github.com/user-attachments/assets/f35bad72-1537-442b-b18d-c169dedbafed)

- 이러한 이유로 GlobalScope는 @DelicateCoroutinesApi와 함께 선언된다
- GlobalScope에 대한 경고라고 생각하면 된다

### 1-4 코루틴 콘텍스트와 구조화된 동시성
- 코루틴 콘텍스는 구조화된 동시성 개념과 밀접한 관련이 있음
- 새로운 코루틴을 시작할때 자식 코루틴은 부모 콘텍스트를 상속받는다
- 그런 다음 새로운 코루틴은 부모-자식 관계를 설정하는 역할을 하는 새 Job 객체를 생성한다
- 마지막으로 코루틴 콘텍스트에 전달된 인자가 적용된다
```.kt
fun main() {
   runBlocking(Dispatchers.Default) {
       log(coroutineContext)
       launch {
           log(coroutineContext)
           launch(Dispatchers.IO + CoroutineName("mine")) {
               log(coroutineContext)
           }
       }
   }
}

// 0 [DefaultDispatcher-worker-1 @coroutine#1] [CoroutineId(1),
   "coroutine#1":BlockingCoroutine{Active}@68308697, Dispatchers.Default]
// 1 [DefaultDispatcher-worker-2 @coroutine#2] [CoroutineId(2),
   "coroutine#2":StandaloneCoroutine{Active}@2b3ce773, Dispatchers.Default]
// 2 [DefaultDispatcher-worker-3 @mine#3] [CoroutineName(mine),
   CoroutineId(3), "mine#3":StandaloneCoroutine{Active}@7c42841a,
   Dispatchers.IO]
```
- 이를 알았으면 새로운 코루틴이 부모 코루틴이 사용한 디스패처를 그대로 사용한다는 것을 알 수 있다
- ![image](https://github.com/user-attachments/assets/705690a6-ca8f-4060-bd3c-9ed76bb19e32)

- runBlocking은 특수한 디스패처인 BlockingEventLoop로 시작되며 BlockingCoroutine이라는 Job 객체를 생성한다
- launch는 Default Dispatcher를 상속받고 자신의 Job 객체로 StandaloneCoroutine을 생성한다

- 코드에서 코루틴 부모-자식 관계, 코루틴과 연관된 Job의 관계를 실제로 확인할 수 있다
```.kt
import kotlinx.coroutines.job


fun main() = runBlocking(CoroutineName("A")) {
   log("A's job: ${coroutineContext.job}")
   launch(CoroutineName("B")) {
      log("B's job: ${coroutineContext.job}")
      log("B's parent: ${coroutineContext.job.parent}")
   }
   log("A's children: ${coroutineContext.job.children.toList()}")
}


// 0 [main @A#1] A's job: "A#1":BlockingCoroutine{Active}@41
// 10 [main @A#1] A's children: ["B#2":StandaloneCoroutine{Active}@24
// 11 [main @B#2] B's job: "B#2":StandaloneCoroutine{Active}@24
// 11 [main @B#2] B's parent: "A#1":BlockingCoroutine{Completing}@41
```
- launch, async와 같은 코루틴 빌더 함수로 시작된 코루틴과 마찬가지로 coroutineScope함수도 자체 Job객체를 갖고 계층 구조에 참여한다

```.kt
fun main() = runBlocking<Unit> { // coroutine#1
   log("A's job: ${coroutineContext.job}")
   coroutineScope {
       log("B's parent: ${coroutineContext.job.parent}") // A
       log("B's job: ${coroutineContext.job}") // C
       launch { //coroutine#2
           log("C's parent: ${coroutineContext.job.parent}") // B
       }
   }
}


// 0 [main @coroutine#1] A's job: "coroutine#1":BlockingCoroutine{Active}@41
// 2 [main @coroutine#1] B's parent:
   "coroutine#1":BlockingCoroutine{Active}@41
// 2 [main @coroutine#1] B's job: "coroutine#1":ScopeCoroutine{Active}@56
// 4 [main @coroutine#2] C's parent:
   "coroutine#1":ScopeCoroutine{Completing}@56
```

## 2. 취소
- 코드가 완료되기 전에 취소하는 것은 매우 중요하다
- 취소는 불필요한 작업을 막아주며
- 메모리나 리소스 누수를 방지하는 데도 도움을 준다
- 오류 처리에도 중요한 역할을 한다

### 2-1 취소 촉발
- 앞에서 본 Job을 return하는 launch와 Deferred를 return하는 async 모두 cancel 메서드를 사용할 수 있다
```.kt
fun main() {
    runBlocking {
        val launchedJob = launch {
            log("I'm launched!")
            delay(1000.milliseconds)
            log("I'm done!")
        }
        val asyncDeferred = async {
            log("I'm async")
            delay(1000.milliseconds)
            log("I'm done!")
        }
        delay(200.milliseconds)
        launchedJob.cancel()
        asyncDeferred.cancel()
    }
}
 
// 0 [main @coroutine#2] I'm launched!
// 7 [main @coroutine#3] I'm async
```

### 2-2 시간제한이 초과된 후 자동으로 취소 호출
- `withTimeout`, `withTimeoutOrNull`함수를 사용하면 시간을 제한하면서 값을 계산할 수 있게 해준다
- `withTimeout`함수는 타임아웃이 되면 예외(TimeoutCancellationException)을 발생시킨다
- 만약 잡아내려면 try-catch 블록으로 잡아야 한다
- `withTimeoutOrNull` 함수는 타임아웃이 발생하면 null을 리턴한다
#### Note
- `withTimeout`이 발생시키는 예외는 CancellationException으로 코루틴을 취소시킨다
- 이는 의도와 다르게 모든 코루틴이 취소될 수 있으므로 `withTimeoutOrNull`을 사용하는 편이 좋다
####
- 아래 예제는 3초가 걸리는 calculateSomething함수를 호출한다
- 첫 번째 호출에서는 타임아웃이 발생하고 null이 반환된다
- 두 번째 호출에서는 정상적으로 수행된다
```.kt
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.seconds
import kotlin.time.Duration.Companion.milliseconds

suspend fun calculateSomething(): Int {
   delay(3.seconds)
   return 2 + 2
}

fun main() = runBlocking {
   val quickResult = withTimeoutOrNull(500.milliseconds) {
       calculateSomething()
   }
   println(quickResult)
   // null
   val slowResult = withTimeoutOrNull(5.seconds) {
       calculateSomething()
   }
   println(slowResult)
   // 4
}
```
- ![image](https://github.com/user-attachments/assets/5a18d975-d115-4b1e-a12c-5efc4ab14878)


### 2-3 취소는 모든 자식 코루틴에게 전파된다
- 구조화된 동시성의 강력한 기능으로, 코루틴을 취소하면 해당 코루틴의 모든 자식 코루틴도 자동으로 취소된다
```.kt
fun main() = runBlocking {
    val job = launch {
        launch {
            launch {
                launch {
                    log("I'm started")
                    delay(500.milliseconds)
                    log("I'm done!")
                }
            }
        }
    }
    delay(200.milliseconds)
    job.cancel()
}
 
// 0 [main @coroutine#5] I'm started
```
- 코루틴은 어디에서 어떻게 취소될 수 있는가?

### 2-4 취소된 코루틴은 특별한 지점에서 CancellationException을 던진다
- 취소 메커니즘은 CancellationException이라는 특수한 예외를 던진다
- 우선적으로는 일시 중단 지점이 이 예외를 던지는 지점이다
- 아래 예시에서 영역이 취소되었는지 여부에 따라 'A'나 'ABC'가 출력되며 'AB'는 절대 출력되지 않는다. 
- 'B'와 'C'사이에 취소 지점이 없기 때문이다
```.kt
coroutineScope {
    log("A")
    delay(500.milliseconds)
    log("B")
    log("C")
}
```
- 코루틴은 예외를 사용해 코루틴 계층에서 취소를 전파하기 때문에 이 예외를 실수로 삼키거나 직접 처리하지 않도록 주의해야 한다
```.kt
suspend fun doWork() {
    delay(500.milliseconds)
    throw UnsupportedOperationException("Didn't work!")
}
 
fun main() {
    runBlocking {
        withTimeoutOrNull(2.seconds) {
            while (true) {
                try {
                    doWork()
                } catch (e: Exception) { // 여기서 예외를 잡아 취소를 막는다
                    println("Oops: ${e.message}")
                }
            }
        }
    }
}
 
// Oops: Didn't work!
// Oops: Didn't work!
// Oops: Didn't work!
// Oops: Timed out waiting for 2000 ms
// ... (does not terminate)
```
- 우리는 취소가 전파되기를 원하지만 catch 에서 모든 예외를 잡고 있어 이 코드는 무한히 반복된다
- 이를 막으려면 if(e is CancellationException) throw e와 같이 예외를 던지거나
- 처음부터 UnsupportedOperationException만 잡아야 한다

### 2-5 취소는 협력적이다
- 직접 작성한 코드에서 코루틴을 취소 가능하게 만들어야 한다. 아래 코드가 취소 가능하다고 생각할 수 있지만 실제로 코드를 보자
```.kt
suspend fun doCpuHeavyWork(): Int {
    log("I'm doing work!")
    var counter = 0
    val startTime = System.currentTimeMillis()
    while (System.currentTimeMillis() < startTime + 500) {
        counter++
    }
    return counter
}
 
fun main() {
    runBlocking {
        val myJob = launch {
            repeat(5) {
                doCpuHeavyWork()
            }
        }
        delay(600.milliseconds)
        myJob.cancel()
    }
}
```
- 이 코드가 2번 "I'm doing work" 메시지를 출력한 후 취소되리라 예상할 수도 있지만 실제 출력은 프로그램이 종료되기 전에 doCpuHeavyWork 함수가 5번 모두 완료된다는 사실을 보여준다
```.kt
30 [main @coroutine#2] I'm doing work!
535 [main @coroutine#2] I'm doing work!
1036 [main @coroutine#2] I'm doing work!
1537 [main @coroutine#2] I'm doing work!
2042 [main @coroutine#2] I'm doing work!
```

- 함수가 취소되려면 일시 중단 지점이 있어야 하는데, 위 코드에서는 존재하지 않는다
- 따라서 일시 중단 함수는 스스로 취소 가능하게 로직을 제공해야 한다
- 코드가 취소 가능한 다른 함수를 호출할 때는 자동으로 취소 가능 지점이 도입된다
- 예를 들어 doWork 함수 본문에 delay 호출을 추가하면 해당 함수는 취소 가능한 지점을 갖게 된다
```.kt
suspend fun doCpuHeavyWork(): Int {
    log("I'm doing work!")
    var counter = 0
    val startTime = System.currentTimeMillis()
    while (System.currentTimeMillis() < startTime + 500) {
        counter++
        delay(100.milliseconds)
    }
    return counter
}
```
- delay가 추가되었으므로 취소할 수 있는 지점이 생긴다
- 이외에도 ensureActive, yield, isActive 속성이 있다

### 2-6 코루틴이 취소됐는지 확인
- 코루틴이 취소됐는지 확인하려면 CoroutineScope의 isActive 속성을 확인한다
```.kt
val myJob = launch {
   repeat(5) {
       doCpuHeavyWork()
       if(!isActive) return@launch
   }
}
```
- 이 동작을 하는 편의 함수로 ensureActive를 제공한다. 이 함수는 코루틴이 더 이상 활성 상태가 아닐 경우, CancellationException을 던진다
```.kt
val myJob = launch {
   repeat(5) {
       doCpuHeavyWork()
       ensureActive()
   }
}
```

### 2-7 다른 코루틴에게 기회를 주기: yield 함수
- 이와 관련해 yield라는 함수도 제공한다.
- 이 함수는 코드 안에서 취소 가능 지점을 제공할 뿐만 아니라 현재 점유된 디스패처에서 다른 코루틴이 작업할 수 있게 해준다
```.kt
import kotlinx.coroutines.*

fun doCpuHeavyWork(): Int {
   var counter = 0
   val startTime = System.currentTimeMillis()
   while (System.currentTimeMillis() < startTime + 500) {
       counter++
   }
   return counter
}

fun main() {
   runBlocking {
       launch {
           repeat(3) {
               doCpuHeavyWork()
           }
       }
       launch {
           repeat(3) {
               doCpuHeavyWork()
           }
       }
   }
}
```
- doCpuHeavyWork 함수가 일시 중단 지점을 포함하지 않으면 첫 번째로 실행된 코루틴이 완료될 때 까지 두 번째 코루틴은 실행되지 않음을 알 수 있다
```.kt
29 [main @coroutine#2] I'm doing work!
533 [main @coroutine#2] I'm doing work!
1036 [main @coroutine#2] I'm doing work!
1537 [main @coroutine#3] I'm doing work!
2042 [main @coroutine#3] I'm doing work!
2543 [main @coroutine#3] I'm doing work!
```

- 그 이유는 코루틴 본문에 중단점이 없기 때문에 첫 번째 코루틴이 일시중단 될 기회가 없어 두 번째 코루틴이 실행되지 못한다
- isActive를 확인하거나 ensureActive를 호출해도 상황은 바뀌지 않는다
- 이 함수들은 취소 여부만 확인할 뿐 실제로 코루틴을 일시 중단시키지 않는다
- 여기서 yield함수가 유용하다. 이 함수는 코드에서 CancellationException을 던질 수 있는 지점을 제공하며 다른 코루틴에게 제어를 넘길 수 있는 중단점을 제공한다
```.kt
suspend fun doCpuHeavyWork(): Int {
   var counter = 0
   val startTime = System.currentTimeMillis()
   while (System.currentTimeMillis() < startTime + 500) {
       counter++
       yield()
   }
   return counter
}

0 [main @coroutine#2] I'm doing work!
559 [main @coroutine#3] I'm doing work!
1062 [main @coroutine#2] I'm doing work!
1634 [main @coroutine#3] I'm doing work!
2208 [main @coroutine#2] I'm doing work!
2734 [main @coroutine#3] I'm doing work!
```
- 그림 15.6은 일시 중단 지점과 취소 지점이 없는경우 isActive, ensureActive를 사용하는 것과 yield를 사용하는 것의 차이를 보여준다
- ![image](https://github.com/user-attachments/assets/1993d7e9-1023-4865-8628-06048439e300)


### 2-8. 리소스를 얻을 때 취소를 염두에 두기
- DB, 네트워크 IO 등과 같은 리소스를 사용해 작업해야 할때 이를 명시적으로 닫아야 적절하게 해제된다
- 취소는 다른 예외와 마찬가지로 함수의 조기 반환을 유발할 수 있으므로 finally를 통해 적절히 처리해야 한다
```.kt
class DatabaseConnection : AutoCloseable {
   fun write(s: String) = println("writing $s!")
   override fun close() {
       println("Closing!")
   }
}

fun main() {
   runBlocking {
       val dbTask = launch {
           val db = DatabaseConnection()
           delay(500.milliseconds)
           db.write("I love coroutines!")
           db.close()
       }
       delay(200.milliseconds)
       dbTask.cancel()
   }
   println("I leaked a resource!")
}
```
- 이 코드는 데이터베이스를 닫기 전에 코루틴이 취소되어 close가 호출되지 못했다

```.kt
val dbTask = launch {
   val db = DatabaseConnection()
   try {
       delay(500.milliseconds)
       db.write("I love coroutines!")
   } finally {
       db.close()
   }
}
```
- 코루틴 내에서도 try - finally 블록을 사용할 수 있다
```.kt
val dbTask = launch {
   DatabaseConnection().use {
       delay(500.milliseconds)
       it.write("I love coroutines!")
   }
}
```
- AutoClosable 인터페이스를 구현하는 경우, 10장에서 배운 use함수를 사용해 더 간결하게 처리할 수 있다

### 2-9 프레임워크가 여러분 대신 취소를 할 수 있다
- 많은 실제 프레임워크에서는 적절한 코루틴 스코프를 제공하고 취소를 자동으로 처리한다
- 안드로이드에선 ViewModel 클래스가 viewModelScope를 제공한다
```.kt
class MyViewModel: ViewModel() {
    init {
        viewModelScope.launch {
            while (true) {
                println("Tick!")
                delay(1000.milliseconds)
            }
        }
    }
}
```
- 다른 예제로 Ktor를 사용한 서버 측 애플리케이션이 있다.
- 여기서 각 요청 핸들러는 암시적 수신자로 PipelineContext 객체를 가지고 있는데, 이 객체는 CoroutineScope를 상속한다
```.kt
routing {
    get("/") { // this: PipelineContext
        launch {
            println("I'm doing some background work!")
            delay(5000.milliseconds)
            println("I'm done")
        }
    }
}
```
```.kt
routing {
    get("/") {
        call.application.launch {
            println("I'm doing some background work!")
            delay(5000.milliseconds)
            println("I'm done")
        }
    }
}
```
