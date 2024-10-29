# Flow 는 무엇일까 👀

코틀린 `Flow` 에 대한 정의를 찾아보니 다음과 같이 적혀있었다.

> An asynchronous data stream that sequentially emits values and completes normally or with an exception.
>

글을 읽어보니 **비동기(asynchronous)** 와  **데이터 스트림(data stream)** 이라는 키워드가 핵심인 것 같았다. 
'비동기'라는 단어는 많이 들어봤어도, '데이터 스트림'은 너무 생소했다. 일단 데이터 스트림이 무엇인지부터 알아보자.

### 📌 데이터 스트림

데이터 스트림(Data Stream)은 **데이터가 순차적으로 흐르는 흐름**을 의미한다. 일반적으로 데이터가 시간에 따라 순차적으로 생산되고 소비된다는 특징을 가지고 있다.


#### Producer 와 Consumer

데이터 스트림에서 중요한 역할을 하는 두 가지 구성 요소가 있다. 바로 생산자 (Producer)와 소비자 (Consumer) 이다.

- **생산자**는 데이터를 생성하는 주체로, API 서버, 센서, 데이터베이스 등을 예시로 들 수 있다. 데이터 스트림에 새로운 데이터를 추가하거나 이벤트를 발생시킨다.
- **소비자**는 생산자가 생성한 데이터를 소비하는 주체를 의미한다. 데이터 처리를 수행하는 클라이언트, UI 등이 포함된다.

![image2.png](img%2Fimage2.png)

<br>

#### Hot Stream 과 Cold Stream

데이터 스트림은 데이터 생산 및 소비 방식에 따라 Hot 과 Cold 로 구분할 수 있다.

- **Hot Stream** : 데이터를 소비하는 것과 무관하게 원소를 생성한다. 소비자가 구독하기 전에 이미 데이터가 생산되고 있으며, 구독자가 데이터를 요청하면 최신 데이터를 수신하게 된다.
- **Cold Stream** : 데이터 요청이 있을 때만 작업을 수행한다. 소비자가 데이터 스트림에 연결할 때 데이터 생산이 시작된다. 해당 방식은 각각의 소비자에게 독립적인 데이터를 제공한다.

이러한 데이터 스트림의 특성으로 인해 Flow는 일련의 값을 순차적으로 발행(emit)하고, 소비자는 이를 수집(collect)하여 사용할 수 있다. 
또한 Flow는 Cold 데이터 스트림에 속하므로 소비자가 구독하기 전까지는 시작되지 않는다는 특징을 가지고 있다.

![image1.webp](img%2Fimage1.webp)


<br>

다음 코드를 보면서 이해해보자.

```kotlin
fun simple(): Flow<Int> = flow { 
    println("Flow started")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    println("Calling simple function...")
    val flow = simple()
    println("Calling collect...")
    flow.collect { value -> println(value) } 
    println("Calling collect again...")
    flow.collect { value -> println(value) } 
}
```

`Flow<Int>` 타입을 반환하는 `simple()` 함수는 생성자 역할을 하며, emit 을 통해 `Int` 값을 발행한다. 
`main()` 함수에서는 `collect`를 통해 생산자로부터 흐름을 수집하고 있다. 이때 생산자가 `emit`을 통해 발행하는 각 값들을 받을 수 있다.


호출 결과는 다음과 같다.
```kotlin
Calling simple function...
Calling collect...
Flow started
1
2
3
Calling collect again...
Flow started
1
2
3
```

일반적인 예상과 다르게`simple()` 함수가 호출되는 시점에서 "Flow started" 가 출력되지 않는다.
이는 Cold 스트림의 특징을 아주 잘 보여준다. 

"Flow started" 는 `collect` 메서드를 호출할 때 출력 된다.
이를 통해 소비자가 collect 로 데이터를 수집하기 전까지는 데이터를 생산하지 않는다는 것을 알 수 있다.

또한 Flow를 collect 할 때마다 "Flow started" 가 동일하게 출력되는 것을 보아, 매번 Flow가 새로 시작되었음을 의미한다는 것을 알 수 있다. 이러한 특성 덕분에 각 소비자는 Flow를 수집하는 시점에서 새로운 데이터 스트림을 받게 된다.


### 📌 비동기

다음은 **비동기(asynchronous)** 키워드에 초점을 맞춰보려고 한다.

```kotlin
fun simple(): Flow<Int> = flow { // flow builder
    for (i in 1..3) {
        delay(100) // pretend we are doing something useful here
        emit(i) 
    }
}

fun main() = runBlocking<Unit> {
    // Launch a concurrent coroutine to check if the main thread is blocked
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(100)
        }
    }
    // Collect the flow
    val flow = simple()
    flow.collect { value -> println(value) } 
}
```

Flow는 `flow {}`와 같은 빌더 함수를 통해 생성할 수 있다. 빌더 내의 코드를 보면 익숙한 `delay` 메서드가 보인다. (`delay` 는 유명한 suspend 함수이라는 것을 알고 있을 것이다.)

Flow 빌더 내에서 `suspend` 함수를 호출할 수 있다는 의미는 Flow를 정의하는 과정에서 비동기 작업을 쉽게 수행할 수 있다는 것을 나타낸다.  실제 사용시에는 네트워크 요청, 또는 다른 장시간 수행되는 작업들을 메인 스레드를 차단하지 않고도 수행할 수 있음을 의미한다.

코틀린에서는 비동기 작업을 수행하는 함수에 일반적으로 `suspend` 수정자가 붙는다는 것을 알 수 있다. 그러나 위의 코드에서 `simple` 함수는 `suspend`로 표시되지 않았지만, 비동기 작업을 수행하고 있는`Flow`를 반환하고 있다.

이 의미는 `simple()` 함수가 호출될 때, 호출자는 즉시 반환되며 중단되지 않음을 뜻한다. 즉, Flow 자체가 실행되기 전까지는 어떤 비동기 작업도 수행되지 않는다.

Flow의 값은 `collect` 함수를 호출할 때 실제로 수집되며, 이 시점에서 Flow가 비로소 시작된다. 따라서 `collect` 는 `suspend` 함수로 구현되어 Flow에서 값을 수집하기 위한 비동기 작업을 수행한다. 
결과적으로, 비동기적으로 흐름을 수집함으로써 UI 스레드를 차단하지 않고도 네트워크 요청이나 데이터 처리와 같은 긴 작업을 효과적으로 관리할 수 있다!


호출 결과는 다음과 같다.

```kotlin
I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
```

`simple()`에서 생성된 Flow를 비동기적으로 수집하고 있다는 것을 알 수 있다.


또한 Flow는 Kotlin의 코루틴과 결합하여 비동기 데이터를 유연하게 관리할 수 있는 강력한 도구이다.
따라서 Flow는 코루틴의 구조적 동시성(structured concurrency)을 기반으로 동작한다.
코루틴 스코프는 코루틴의 생명 주기와 실행 컨텍스트를 정의하며, 이를 통해 특정 스코프 내에서 실행되는 모든 코루틴이 해당 스코프의 컨텍스트(예: Dispatcher, Job)를 공유하게 된다.
이 컨텍스트는 스코프 내의 코루틴들이 동일한 구조적 제약을 따라 실행되고 종료되도록 보장하여, 코드의 흐름을 예측 가능하게 하고 자원을 효율적으로 관리할 수 있게 한다.

이러한 구조적 동시성은 `Flow`의 비동기 작업들이 상위 스코프의 생명 주기에 맞춰 자동으로 취소되거나 재시작되도록 한다. 예를 들어, `Flow.collect`를 특정 스코프에서 실행하면, 상위 스코프가 종료될 때 `collect` 작업도 함께 취소된다. 이러한 특성은 메모리 누수를 방지하고, 자원의 효율적 관리와 예외 처리를 수월하게 하여 안정적인 비동기 작업을 구현할 수 있게 한다.

### 📌 Flow의 구성 요소

`Flow`는 생성자, 중간 연산자, 최종 연산자로 구성되어 있으며, 다양한 연산자와 결합해 복잡한 비동기 데이터 처리도 간결하고 효율적으로 처리할 수 있다.

먼저, 생성자는 `Flow`를 생성하는 역할을 한다. 데이터를 생성하고 방출하는 첫 번째 단계로, 일반적으로 `flowOf`, `flow`, `asFlow`와 같은 함수를 사용하여 생성한다.

```kotlin
val flow = flowOf(1, 2, 3, 4, 5) 
```

중간 연산자는 생성된 `Flow`에 대해 데이터를 변환하거나 필터링하는 작업을 수행한다. 이러한 연산자는 불변성을 가지며, 원래의 `Flow`를 변경하지 않고 새로운 `Flow`를 반환한다. 중간 연산자는 `map`, `filter`, `flatMap`, `take`와 같은 함수를 포함한다.

```kotlin
val flow = flowOf(1, 2, 3)
    .map { it + 1 }    
    .filter { it == 3 }
```

위의 코드에서 `map` 연산자를 통해 각 요소에 1을 더한 후, 그 결과가 3인 요소만 남기는 `filter` 연산자를 적용하였다.
이처럼 중간 연산자들을 체이닝하여 데이터 스트림을 단계적으로 가공할 수 있다.

또한, 중간 연산자인 `flowOn`을 활용하여 `Flow`의 `CoroutineContext`를 변경할 수 있다. `flowOn`은 상류 흐름의 `CoroutineContext`를 변경하며, 이는 프로듀서와 `flowOn` 이전에 적용된 모든 중간 연산자에 영향을 미친다.
그러나 `flowOn` 이후의 하류 흐름(즉, `flowOn` 이후의 중간 연산자와 소비자)은 영향을 받지 않으며, `Flow`를 수집하는 데 사용된 `CoroutineContext`에서 실행된다.

이러한 방식으로 중간 연산자들을 적절히 조합하면, 데이터 흐름을 효율적이고 직관적으로 관리할 수 있다.

최종 연산자는 `Flow`의 실행을 완료하고 실제로 데이터를 수집하거나 방출하는 역할을 한다.
이 단계에서 데이터 흐름이 시작되며, 최종 연산자가 호출되지 않으면 중간 연산자에서 정의된 데이터 흐름이 실행되지 않는다. `collect`, `single`, `toList`와 같은 함수를 사용하여 데이터를 수집한다.

<br>

Flow의 가장 큰 장점은 Cold 스트림 특성 덕분에 소비자가 collect를 호출하기 전까지 데이터 생산이 이루어지지 않는다는 점이다.
이는 메모리 사용을 최적화하고, 필요할 때만 데이터를 처리할 수 있게 해준다.
또한, 비동기적 특성으로 인해 UI 스레드를 차단하지 않으면서 네트워크 요청이나 긴 작업을 처리할 수 있어, 사용자 경험을 향상시키는 데 기여한다.
또한, 생성자, 중간 연산자, 최종 연산자를 활용하여 복잡한 비동기 데이터 흐름을 간결하고 직관적으로 다룰 수 있다. 이로써 개발자는 데이터의 변환 및 필터링을 손쉽게 수행할 수 있으며, 코루틴의 구조적 동시성을 통해 안정적인 비동기 작업을 구현할 수 있다.
이러한 장점들을 가진 Flow를 활용하여 효과적인 비동기 프로그램을 만들어보자!
