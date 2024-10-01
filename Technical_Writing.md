# Flow 는 무엇일까 👀

코틀린 `Flow` 에 대한 정의를 찾아보니 다음과 같이 적혀있었다.

> An asynchronous data stream that sequentially emits values and completes normally or with an exception.
>

글을 읽어보니 **비동기(asynchronous)** 와  **데이터 스트림(data stream)** 이라는 키워드가 핵심인 것 같았다. 비동기는 많이 들어봤어도, 데이터 스트림이라는 단어는 너무 생소했다. 일단 데이터 스트림이 무엇인지부터 알아보자.

### 📌 데이터 스트림

데이터 스트림(Data Stream)은 **데이터가 순차적으로 흐르는 흐름**을 의미한다. 일반적으로 데이터가 시간에 따라 순차적으로 생산되고 소비된다는 특징을 가지고 있다.


#### Producer 와 Consumer

데이터 스트림에서 중요한 역할을 하는 두 가지 구성 요소가 있다. 바로 생산자(Producer)와 소비자(Consumer) 이다.

- **생산자**는 데이터를 생성하는 주체로, API 서버, 센서, 데이터베이스 등을 예시로 들 수 있다. 데이터 스트림에 새로운 데이터를 추가하거나 이벤트를 발생시킨다.
- **소비자**는 생산자가 생성한 데이터를 소비하는 주체를 의미한다. 데이터 처리를 수행하는 클라이언트, UI 등이 포함된다.

![image2.png](img%2Fimage2.png)

<br>

#### Hot Stream 과 Cold Stream

데이터 스트림은 데이터 생산 및 소비의 방식에 따라 Hot 과 Cold 로 구분할 수 있다.

- **Hot Stream** : 데이터를 소비하는 것과 무관하게 원소를 생성한다. 소비자가 구독하기 전에 이미 데이터가 생산되고 있으며, 구독자가 데이터를 요청하면 최신 데이터를 수신하게 된다.
- **Cold Stream** : 데이터 요청이 있을 때만 작업을 수행한다. 소비자가 데이터 스트림에 연결할 때 데이터 생산이 시작된다. 해당 방식은 각각의 소비자에게 독립적인 데이터를 제공한다.

이러한 데이터 스트림의 특성으로 인해 Flow는 일련의 값을 순차적으로 발행(emit)하고, 소비자는 이를 수집(collect)하여 사용할 수 있다. 또한 Flow는 Cold 데이터 스트림에 속하므로 소비자가 구독하기 전까지는 시작되지 않는다.

![image1.webp](img%2Fimage1.webp)


<br>

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

위의 코드를 보면서 이해해보자.

`Flow<Int>` 타입을 반환하는 `simple()` 함수는 생성자 역할을 하며, emit 을 통해 `Int` 값을 발행한다.  `main()` 함수에서는 `collect`를 통해 생산자로부터 흐름을 수집하고 있다. 이때 생산자가 `emit`을 통해 발행하는 각 값들을 받을 수 있다.


#### 호출 결과
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

`simple()` 함수가 호출되었을 때 예상과는 다르게  "Flow started" 가 바로 출력되지 않는다. 이는 Cold 스트림의 특징을 아주 잘 보여준다. 

"Flow started" 는 collect 메서드를 호출할 때 출력 된다. 소비자가 collect 로 데이터를 수집하기 전까지는 데이터를 생산하지 않는다는 것을 알 수 있다.

또한 Flow를  collect 할 때마다 "Flow started" 가 동일하게 출력되는 것을 보아, Flow가 새로 시작되었음을 의미한다는 것을 알 수 있다. 이러한 특성 덕분에 각 소비자는 Flow를 수집하는 시점에서 새로운 데이터 스트림을 받게 된다.

<br>

### 📌 비동기

다음은 **비동기(asynchronous)** 키워드에 초점을 맞춰보려고 한다.

```kotlin
fun simple(): Flow<Int> = flow { // flow builder
    for (i in 1..3) {
        delay(100) // pretend we are doing something useful here
        emit(i) // emit next value
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

Flow 빌더 내에서 `suspend` 함수를 호출할 수 있다는 의미는 Flow를 정의하는 과정에서 비동기 작업을 쉽게 수행할 수 있다는 것을 나타낸다.  실제 사용시에는 네트워크 요청, 또는 다른 장시간 수행되는 작업을 사용하여 메인 스레드를 차단하지 않고도 수행할 수 있음을 의미한다.

코틀린에서는 비동기 작업을 수행하는 함수에 일반적으로 `suspend` 수정자가 붙는다는 것을 알 수 있다. 그러나 위의 코드에서 `simple` 함수는 `suspend`로 표시되지 않았지만, 비동기 작업을 수행하고 있는`Flow`를 반환하고 있다.

이 의미는 `simple()` 함수가 호출될 때, 호출자는 즉시 반환되며 중단되지 않음을 뜻한다. 즉, Flow 자체가 생성되기까지는 어떤 비동기 작업도 수행되지 않는다.

앞서 언급했듯이, Flow의 값은 `collect` 함수를 호출할 때 실제로 수집되며, 이 시점에서 Flow가 비로소 시작된다. 따라서 collect 는 suspend 함수로 구현되어 Flow에서 값을 수집하기 위한 비동기 작업을 수행한다. 결과적으로, 비동기적으로 흐름을 수집함으로써, UI 스레드를 차단하지 않고도 네트워크 요청이나 데이터 처리와 같은 긴 작업을 효과적으로 관리할 수 있다!

(위의 코드는 `runBlocking` 스코프 내에서 `collect`를 호출했기 때문에, 별도로 `launch`를 사용하지 않은 것이다.)

#### 호출 결과

```kotlin
I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
```

코드 실행 결과를 보면 `simple()`에서 생성된 Flow를 비동기적으로 수집하고 있다는 것을 알 수 있다.

<br>
Flow는 Kotlin의 코루틴과 결합하여 비동기 데이터를 유연하게 관리할 수 있는 강력한 도구다. 
Flow를 사용하면 데이터 스트림을 필요할 때만 생성하는 Cold 스트림의 이점을 활용할 수 있으며, 해당 글에서 다루지는 못했지만 다양한 연산자와 결합해 복잡한 데이터 처리도 간결하고 효율적으로 처리할 수 있다.

간단하게나마 언급해보자면, Flow는 비동기 처리에서 중간 연산자(intermediate operators)와 최종 연산자(terminal operators)를 통해 데이터를 변환하거나 필터링할 수 있다. 
예를 들어, `map`, `filter`, `take`, `combine` 등의 중간 연산자를 사용해 데이터 스트림을 여러 단계에 걸쳐 가공하고, 최종적으로 `collect`와 같은 최종 연산자를 통해 수집할 수 있다. 
이 과정에서 비동기 작업을 손쉽게 처리할 수 있어 네트워크 요청, 파일 읽기, 타이머 등 긴 시간이 필요한 작업을 자연스럽게 수행할 수 있다.

또한, Flow는 코루틴의 구조적 동시성(structured concurrency)을 기반으로 동작하기 때문에, 비동기 작업의 수명 주기를 명확히 관리할 수 있다. 
이로 인해, 메모리 누수 없이 안정적인 비동기 작업을 구현할 수 있고, 예외 처리도 쉽다. 
Flow가 제공하는 `catch`나 `onCompletion`과 같은 연산자는 데이터 흐름 내에서 발생하는 오류를 효과적으로 처리할 수 있도록 돕는다.

