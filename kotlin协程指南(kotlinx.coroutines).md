# kotlinx.coroutines 示例指南

这是一个简短的关于 `kotlinx.coroutines` 的指导，并且包含一系列示例。

## 介绍和设置

Kotlin，作为一种编程语言，仅仅在标准库中提供了最小化的低层次的API，以使得其他的第三方库去利用协程。与其他的语言特性不同，在Kotlin中， `async` 和 `await` 并不是它的关键字，甚至不是它的标准库的一部分。

`kotlinx.coroutines` 是一个内容丰富的的类库，它包含了大量的原生支持协程的高级API，同样也是这篇指导将会提及的， 包括 `async` 和 `await`，为了能根据此篇指导使用原生的协程功能，你需要在 `kotlinx-coroutines-core` 模块中增加相应的依赖以对其进行扩展。

## 目录

[TOC]

## 协程的基本内容
这个章节将会介绍一些关于协程的基本概念。

### 第一个协程程序

执行以下代码：

```Java
fun main(args: Array<String>) {
    launch(CommonPool) { // create new coroutine in common thread pool
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello,") // main function continues while coroutine is delayed
    Thread.sleep(2000L) // block main thread for 2 seconds to keep JVM alive
}
```

> 你可以在[此处](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-01.kt)获得全部的完整代码示例

```java
Hello,
World!
```

本质上来说，协程是一个轻量级的线程。它们通过[launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/launch.html)协程构造器进行启动，你可以使用`thread { ... }` 代替上例中的 `launch(CommonPool) { ... }` ，使用 `Thread.sleep(...)` 代替上例中的 `delay(...)`。尝试一下。

如果你使用 `thread` 去代替 `launch(CommonPool)` 的时候，编译器将会产生以下错误：
> Error: Kotlin: Suspend functions are only allowed to be called from a coroutine or another suspend function

出现此错误是因为 [delay](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/delay.html)是一个特殊的挂起函数，它不会去阻塞线程，但是，挂起协程的函数只能在一个协程中调用。

### 连接阻塞和非阻塞的世界

在第一个示例中，`main` 函数中混合了非阻塞的 `delay(...)` 函数和阻塞的 `Thread.sleep(...)` 函数。这非常容易混淆让我们使用[runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run-blocking.html)简单的区分阻塞与非阻塞的环境。

```java
fun main(args: Array<String>) = runBlocking<Unit> { // start main coroutine
    launch(CommonPool) { // create new coroutine in common thread pool
        delay(1000L)
        println("World!")
    }
    println("Hello,") // main coroutine continues while child is delayed
    delay(2000L) // non-blocking delay for 2 seconds to keep JVM alive
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-02.kt)找到完整的代码示例

这段代码的结果与上个示例的结果是相同的，但是这段代码仅仅使用了非阻塞的 `delay`

`runBlocking { ... }` 作为一个适配器在顶级的main协程上开始运行。在 `runBlocking` 代码块之外的普通代码会在协程进入 `runBlocking` 代码块内部之前先被执行

以下是以一种同样的方式，使用单元测试实现挂起函数：

```java
class MyTest {
    @Test
    fun testMySuspendingFunction() = runBlocking<Unit> {
        // here we can use suspending functions using any assertion style that we like
    }
}
```

### 等待工作完成

当其他协程正在执行的时候，延迟一段时间进行等待并不是一个很好的实现方式。我们可以明确的指定一直进行等待(非阻塞方式)，直到我们刚刚启动的后台的[Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html)任务完成：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) { // create new coroutine and keep a reference to its Job
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // wait until child coroutine completes
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-03.kt)获取完整的示例代码

现在，最后的结果仍然与之前相同，但此时主协程的代码没有通过任何方式与后台任务进行绑定，比之前的实现更好

### 重构以对功能进行提取

让我们提取 `launch(CommonPool) { ... }` 内部的代码到一个单独的函数中，当你通过重构以后，执行这个“提取出的函数”，你将会得到一个新的修饰符 `suspend`。这是你的第一个挂起函数。
挂起函数能够被协程的内部使用，就像普通的函数一样，但是它们有一些附加的特性，它们可以反过来使用其他的挂起函数，就像 `delay` 在这个例子中一样，暂停执行一个协程。

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) { doWorld() }
    println("Hello,")
    job.join()
}

// this is your first suspending function
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-04.kt)获取完整的示例代码

### 轻量级ARE协程

执行以下代码：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = List(100_000) { // create a lot of coroutines and list their jobs
        launch(CommonPool) {
            delay(1000L)
            print(".")
        }
    }
    jobs.forEach { it.join() } // wait for all jobs to complete
}
```

这段代码开启了10万个协程，1秒钟过后，每一个协程都会打印出一个“.”，现在使用线程改写一下此程序，将会发生什么事情？（大部分情况下你的代码将会产生out-of-memory的错误）

### 协程类似守护线程

以下的代码启动了一个“耗时”的协程操作，此操作会每秒钟打印2次"I'm sleeping"，并且在一定的延时时间之后，返回到主函数中

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    launch(CommonPool) {不会你玩的及诶诶酒店免费检测你禁锢让你父吗 
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // just quit after delay
}
```
> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-basic-06.kt)获取完整的示例代码

你可以执行它，并且能够在终端看到3行输出：
```java
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

活动的协程不会维持处理过程存活，它们就像一个守护线程一样。

## 退出和超时

这个章节包含退出和超时的相关概念

### 退出协程执行

对于一个很小的应用来说，返回“main”方法来含蓄的结束全部的协程可能是一个不错的想法，再大一些，例如耗时的应用，你需要一个更细粒度的控制，“launch”函数返回的“Job”对象能够被用来退出一个正在执行的协程：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to ensure it was cancelled indeed
    println("main: Now I can quit.")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-01.kt)获取完整的示例代码

它将会产生以下输出：

```java
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

一旦我们调用了 `job.cancel`，我们将不会在其他协程上看到任何的输出，因为它们已经退出了。

### 退出是相互合作的

协程的退出是相互合作的，一个协程内部不得不进行合作才能进行退出。所有的挂起函数在 `kotlinx.coroutines` 中都是可退出的。它们会检查协程是否退出，并且当一个协程被取消的时候，它们会抛出[CancellationException](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-cancellation-exception.html)。然而，当一个协程正在进行计算工作并且没有检查退出的话，那么他将不能被退出，就如同下边的例子展示的一样：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        var nextPrintTime = 0L
        var i = 0
        while (i < 10) { // computation loop
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime = currentTime + 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to see if it was cancelled....
    println("main: Now I can quit.")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-02.kt)获取完整的示例代码

运行这段程序，你会发现在调用退出之后，程序依然会打印出"I'm sleeping"

### 使计算代码能够被取消

这里有两个途径能够使计算过程中的代码能够被取消，第一个途 径是调用挂起函数，使用[yield](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/yield.html)函数实现这个目的是一个不错的选择。另外一个方法就是明确的检查取消状态，让我们尝试使用后一种的方案

使用 `while (isActive)` 代替上一个例子中的 `while (true)`，并且重新运行它。

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        var nextPrintTime = 0L
        var i = 0
        while (isActive) { // cancellable computation loop
            val currentTime = System.currentTimeMillis()
            if (currentTime >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime = currentTime + 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to see if it was cancelled....
    println("main: Now I can quit.")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-03.kt)获取完整的示例代码

如你所见，现在这个循环能够被取消，[isActive](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/is-active.html)是通过协程获得的[CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/index.html)对象的内部可用属性

### 使用finally关闭资源

可退出的挂起函数会抛出一个能够使用正常方式处理的[CancellationException](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-cancellation-exception.html)异常

例如 当函数被退出的时候 `try {...} finally {...}` 和Kotlin的 `use` 函数能够正常的执行它们的最终的行为

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            println("I'm running finally")
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to ensure it was cancelled indeed
    println("main: Now I can quit.")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-04.kt)获取完整的示例代码

以上的例子将会产生以下的输出：

```java
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm running finally
main: Now I can quit.
```

### 运行一个不可退出的代码块

像上一个示例中一样，任何尝试在 `finally` 代码块中使用可挂起的函数都将会导致[CancellationException](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-cancellation-exception.html)异常。因为运行这段代码的协程已经被取消了。通常来说，由于良好的关闭行为（关闭一个File，取消一个Job，或者关闭任何一种通讯频道）通常是非阻塞的并且不会涉及到其他的挂起函数，所以这并不是一个问题。
然而，在很少的情况下，当你需要在已经被取消的协程中挂起时，你可以使用[run](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run.html)函数和[NonCancellable](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-non-cancellable/index.html)上下文，将对应的代码包装在 `run(NonCancellable) {...}` 中，如下面的例子展示一样：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = launch(CommonPool) {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            run(NonCancellable) {
                println("I'm running finally")
                delay(1000L)
                println("And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    delay(1300L) // delay a bit to ensure it was cancelled indeed
    println("main: Now I can quit.")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-05.kt)获取完整的示例代码

### 超时

在实际中，最明显的退出协程的原因就是它的执行时间超过了一些时间限制。
当你手动跟踪对应的作业，并启动一个在延迟后取消跟踪的单独的协同程序的时候，有一个[withTimeout](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/with-timeout.html)函数可以供你使用。
请查看以下的示例：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-cancel-06.kt)获取完整的示例代码

这将会产生以下的输出：

```java
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.experimental.TimeoutException: Timed out waiting for 1300 MILLISECONDS
```

[withTimeout](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/with-timeout.html)抛出的 `TimeoutException` 异常是[CancellationException](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-cancellation-exception.html)的私有子类。我们并没有在控制看见它的堆栈跟踪记录，这是因为在已经取消的协程内部， `CancellationException` 被认为是一个协程正常退出的原因，在这里例子中，我们在 `main` 函数中使用了 `withTimeout`。
因为取消仅仅是一个异常，所有的资源都可以被正常的关闭。如果你需要在超时后做一些额外的操作，你可以将你的超时的代码包装在 `try {...} catch (e: CancellationException) {...}` 代码块中。

## 构建挂起函数

此章节将会介绍多种挂起函数的构建方式

### 默认顺序执行

假设此时有2个挂起函数，它们的功能是调用远程服务和计算，这两个挂起函数在其他地方进行定义。在这里我们仅仅假设这两个函数是可用的，但是实际上，这两个函数内部仅仅使用了`delay` 方法进行延迟以达到目的，以下是示例代码：

```java
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```

我们将要如何做，才能使它们按顺序进行调用，首先执行 `doSomethingUsefulOne` 然后执行 `doSomethingUsefulTwo` 如何计算它们返回值的和？实际上，我们使用第一个函数的返回结果，来决定是否需要调用第二个函数或者决定如何调用第二个函数。

我们仅仅使用正常的调用顺序，因为代码在协程中就像在普通代码中一样，是按默认顺序执行的。以下代码演示了测量两个挂起函数执行的总共用时时间。

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = doSomethingUsefulOne()
        val two = doSomethingUsefulTwo()
        println("The answer is ${one + two}")
    }
    println("Completed in $time ms")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-01.kt)获取完整的示例代码

运行代码将会产生以下类似的输出：

```java
The answer is 42
Completed in 2017 ms
```

### 使用async并发

如果上例中，两个调用 `doSomethingUsefulOne` 和 `doSomethingUsefulTwo` 之间并没有依赖关系，当我们想要更快的获得结果，如何做才能使它们共同的并发执行？ [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html)能够帮助我们完成这项工作。

从概念上来讲，[async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html)和[launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/launch.html)是很相似的。它们就像一个轻量级的线程，也就是协程中与其他所有的协程并发的执行。不同的是， `launch` 返回的 `Job` 对象不会携带任何结果数据，而 `async` 将会返回一个[Deferred](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/index.html) -- 这个对象代表一个非阻塞的轻量级的 `Future` 对象，这个对象可以在未来的摸个时间点后获取一个执行结果。你可以在deferred使用 `.await()` 函数来获取它最终的结果。但是，本质上 `Deferred` 仍然是一个 `Job`，所以如果需要的话，你也可以取消它。

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(CommonPool) { doSomethingUsefulOne() }
        val two = async(CommonPool) { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-02.kt)获取完整的示例代码

以上代码将会产生以下类似的输出：

```java
The answer is 42
Completed in 1017 ms
```

这次我们比之前执行快了1倍，因为我们并发的执行了两个协程。注意，我们总是需要明确的指明并发的执行协程。

### 延迟执行async

[CoroutineStart.LAZY](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-start/-l-a-z-y.html)参数能够作为一个参数选项，指定[async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html)延迟执行，只有当这个协程的[await](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-deferred/await.html)或者[start](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/start.html)函数被调用以获取结果的时候，它才会开始执行。运行以下的代码示例，以下代码仅仅与上一个示例有一个参数选项的差别：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(CommonPool, CoroutineStart.LAZY) { doSomethingUsefulOne() }
        val two = async(CommonPool, CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-03.kt)获取完整的示例代码

这会产生以下类似的输出：

```java
The answer is 42
Completed in 2017 ms
```

因此，我们回到了之前的顺序执行，因为我们首先等待 `one` 被调用然后开始执行，然后等待 `two` 被调用在开始执行。这并不是延迟执行的预期的使用场景，它的使用场景是，当计算过程涉及到挂起的功能时，它被用来代替标准库中的 `lazy` 函数。

### 异步风格的函数

当异步调用 `doSomethingUsefulOne` 和 `doSomethingUsefulTwo` 的时候，我们可以使用[async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/async.html)协程构造器来创建异步风格的的函数。
我们可以使用 `async` 前缀或者 `Async` 后缀来定义函数名称，这样做能够突出一个事实，那就是这个函数仅仅会开启一个异步的计算处理，并且需要使用一个 `deferred` 的对象来获取最终的结果，这是一种很好的命名风格。

```java
// The result type of asyncSomethingUsefulOne is Deferred<Int>
fun asyncSomethingUsefulOne() = async(CommonPool) {
    doSomethingUsefulOne()
}

// The result type of asyncSomethingUsefulTwo is Deferred<Int>
fun asyncSomethingUsefulTwo() = async(CommonPool)  {
    doSomethingUsefulTwo()
}
```

值得注意的是， `asyncXXX` 并不是一个挂起函数，它们能够在任何地方被调用。然而，调用它们的时候，也同样意味着异步（这里意味着并发）的执行它们的动作。

以下代码展示了在协程之外使用它们：

```java
// note, that we don't have `runBlocking` to the right of `main` in this example
fun main(args: Array<String>) {
    val time = measureTimeMillis {
        // we can initiate async actions outside of a coroutine
        val one = asyncSomethingUsefulOne()
        val two = asyncSomethingUsefulTwo()
        // but waiting for a result must involve either suspending or blocking.
        // here we use `runBlocking { ... }` to block the main thread while waiting for the result
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-compose-04.kt)获取完整的示例代码

## 协程上下文和调度器

我们已经在之前看见过 `launch(CommonPool) {...}`，`async(CommonPool) {...}`， `run(NonCancellable) {...}`等等，在这些代码片段中，[CommonPool](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-common-pool/index.html)和[NonCancellable](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-non-cancellable/index.html)都是协程的上下文。这个章节将会介绍一些其他的可用选项。

### 调度器和线程

协程调度器决定了当协程执行的时候，使用一个或者多个线程，协程调度器包含在协程的上下文内。协程调度器能够限制一个协程运行在一个特定的线程内，或者将它分发到一个线程池内，或者让它没有线程限制的运行。尝试以下代码：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = arrayListOf<Job>()
    jobs += launch(Unconfined) { // not confined -- will work with main thread
        println(" 'Unconfined': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(context) { // context of the parent, runBlocking coroutine
        println("    'context': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(CommonPool) { // will get dispatched to ForkJoinPool.commonPool (or equivalent)
        println(" 'CommonPool': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs += launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
        println("     'newSTC': I'm working in thread ${Thread.currentThread().name}")
    }
    jobs.forEach { it.join() }
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-context-01.kt)获取完整的示例代码

以上代码将会产生以下输出（可能顺序会稍有不同）：

```java
 'Unconfined': I'm working in thread main
 'CommonPool': I'm working in thread ForkJoinPool.commonPool-worker-1
     'newSTC': I'm working in thread MyOwnThread
    'context': I'm working in thread main
```

[上下文](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/context.html)与[无限制(Unconfined)](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-unconfined/index.html)的上下文的区别将会在后边进行介绍。

### 无限制调度器 vs 有限制调度器

[无限制(Unconfined)](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-unconfined/index.html)协程调度器将会在调用这的线程中开始运行，但是，它仅仅胡运行到第一个挂起点，挂起恢复之后，它运行的线程将完全由被调用的挂起函数所决定。无限制的调度器适用于那些不消耗CPU时间，也不局限与任何特定线程才能更新的共享数据（比如UI）的协程程序。

另一方面，[context](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/context.html)属性可以经由[CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/index.html)接口，在协程的内的部代码块得到，这是一个当前协程的上下文的引用。通过这种方式，父级的上下文就可以被继承。

[runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run-blocking.html)默认的上下文被限制为调用者的线程。所以继承它将会出现FIFO（先进先出）的执行效果。

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val jobs = arrayListOf<Job>()
    jobs += launch(Unconfined) { // not confined -- will work with main thread
        println(" 'Unconfined': I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println(" 'Unconfined': After delay in thread ${Thread.currentThread().name}")
    }
    jobs += launch(context) { // context of the parent, runBlocking coroutine
        println("    'context': I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("    'context': After delay in thread ${Thread.currentThread().name}")
    }
    jobs.forEach { it.join() }
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-context-02.kt)获取完整的示例代码

这将会产生以下输出：

```java
 'Unconfined': I'm working in thread main
    'context': I'm working in thread main
 'Unconfined': After delay in thread kotlinx.coroutines.ScheduledExecutor
    'context': After delay in thread main
```

所以，当没有限制的协程在调用 `delay` 函数后进行恢复之后，使用继承了 `runBlocking {...}` 的上下文的 `context` 的协程，将会继续在 `main` 函数中执行。

### 调试协程和线程

使用[Unconfined](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-unconfined/index.html)调度器或者例如[CommonPool](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlin.coroutines.experimental/-common-pool/index.html)其他的多线程调度器，使得协程能够在一个线程中挂起并且能在其他的线程中恢复，即使使用单线程的调度器，我们也很难找出协程在何时何位置在做什么。通常来说，调试多线程的应用的方法就是在每一个log声明的时候打印线程名称。这个特性在日志框架中是被支持的。单独的线程名称不会给出太多的线程上下文信息，所以 `kotlinx.coroutines` 包括了调试组件使得它更加容易

调试以下代码，并且使用 `-Dkotlinx.coroutines.debug` 的JVM选项

```java
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) = runBlocking<Unit> {
    val a = async(context) {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async(context) {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-context-03.kt)获取完整的示例代码

此处有3个协程，主协程（#1）-- `runBlocking` 和2个计算协程，它们是一个异步deferred对象 `a`（#2）和 `b`（#3）它们全部执行在 `runBlocking` 的上下文内，并且将其限制在了主线程内。输出的内容如下：

```java
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

`log` 函数在方括号中打印线程名称，你可以看见，是主线程的名称，当前执行的协程的标识符被追加到后面，当调试模式打开的时候，标识符会被连续的分配到所有创建的协程中。

你可以获阅读调试组件的[newCoroutineContext](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/new-coroutine-context.html)函数的文档以获得更多关于调试组件的信息。

###  在线程之间进行跳转

使用 `-Dkotlinx.coroutines.debug` JVM参数，运行以下代码

```java
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) {
    val ctx1 = newSingleThreadContext("Ctx1")
    val ctx2 = newSingleThreadContext("Ctx2")
    runBlocking(ctx1) {
        log("Started in ctx1")
        run(ctx2) {
            log("Working in ctx2")
        }
        log("Back to ctx1")
    }
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-context-04.kt)获取完整的示例代码

上边的示例使用了2项新的技术，一个是使用 [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run-blocking.html) 函数明确指定需要的上下文，第二个是使用 [run](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/run.html) 函数改变协程的上下文环境，从下边的输出我们可以看到，改变了运行上下文环境的协程仍然是同一个：

```java
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

### 上下文环境中的Job

[Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html) 是它运行上下文的一部分，在协程中，能够使用 `context[Job]` 表达式，从它的运行上下文中检索它：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    println("My job is ${context[Job]}")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-context-05.kt)获取完整的示例代码

它会产生以下的输出：

```java
My job is BlockingCoroutine{Active}@65ae6ba4
```

所以，在[CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/index.html)中，[isActive](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/is-active.html)仅仅是 `context[Job]!!.isActive` 的一个快捷方式。

### 子协程

当某个协程的 [上下文（context）](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-scope/context.html) 被用于启动其他的协程的时候，新的协程的 [Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html) 就会成为启动新协程的那个协程的Job对象的子Job。当父协程被取消的时候，全部的子协程也都会被递归的取消。

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    // start a coroutine to process some kind of incoming request
    val request = launch(CommonPool) {
        // it spawns two other jobs, one with its separate context
        val job1 = launch(CommonPool) {
            println("job1: I have my own context and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // and the other inherits the parent context
        val job2 = launch(context) {
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
        // request completes when both its sub-jobs complete:
        job1.join()
        job2.join()
    }
    delay(500)
    request.cancel() // cancel processing of the request
    delay(1000) // delay a second to see what happens
    println("main: Who has survived request cancellation?")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-context-06.kt)获取完整的示例代码

产生的输出如下：

```java
job1: I have my own context and execute independently!
job2: I am a child of the request coroutine
job1: I am not affected by cancellation of the request
main: Who has survived request cancellation?
```

### 组合上下文

协程的上下文能够使用 `+` 操作符进行组合，右侧的上下文将会替换左侧上下文的相关条目，例如，一个父协程的[Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html) 能够被继承，而它对应的调度器可以被替换：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    // start a coroutine to process some kind of incoming request
    val request = launch(context) { // use the context of `runBlocking`
        // spawns CPU-intensive child job in CommonPool !!! 
        val job = launch(context + CommonPool) {
            println("job: I am a child of the request coroutine, but with a different dispatcher")
            delay(1000)
            println("job: I will not execute this line if my parent request is cancelled")
        }
        job.join() // request completes when its sub-job completes
    }
    delay(500)
    request.cancel() // cancel processing of the request
    delay(1000) // delay a second to see what happens
    println("main: Who has survived request cancellation?")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-context-07.kt)获取完整的示例代码

程序预期的输出结果如下：

```java
job: I am a child of the request coroutine, but with a different dispatcher
main: Who has survived request cancellation?
```

### 为调试的协程命名

当你需要同一个协程的相关日志记录的时候，自动分配ID是一个不错的选择。然而，当协程绑定在某个特殊的请求，或者做一些特别的后台任务的时候，为了能够达到调试的目的，最好为该协程明确指定一个的名字。[CoroutineName](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-coroutine-name/index.html)与线程名称的函数的作用类似，当调试模式打开的时候，当前正在执行的协程的名称将会将被打印输出。

以下代码将会演示这个概念：

```java
fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main(args: Array<String>) = runBlocking(CoroutineName("main")) {
    log("Started main coroutine")
    // run two background value computations
    val v1 = async(CommonPool + CoroutineName("v1coroutine")) {
        log("Computing v1")
        delay(500)
        252
    }
    val v2 = async(CommonPool + CoroutineName("v2coroutine")) {
        log("Computing v2")
        delay(1000)
        6
    }
    log("The answer for v1 / v2 = ${v1.await() / v2.await()}")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-context-08.kt)获取完整的示例代码

使用 `-Dkotlinx.coroutines.debug` JVM参数运行示例，将会产生以下类似的输出：

```java
[main @main#1] Started main coroutine
[ForkJoinPool.commonPool-worker-1 @v1coroutine#2] Computing v1
[ForkJoinPool.commonPool-worker-2 @v2coroutine#3] Computing v2
[main @main#1] The answer for v1 / v2 = 42
```

### 由明确指定的Job退出

让我们把上下文，父子关系，Job对象的相关知识综合到一起。假设我们的应用中有一个对象，这个对象有它自己的生命周期，但是这个对象并不是一个协程。例如，我们编写一个Android程序，并且，在Android Activity的上下文环境中启动了多种协程，它们负责异步的执行抓取更新数据的操作，或者执行一些动画效果等等，当Activity退出的时候，为了避免内存泄露，我们必须将这些启动的协程取消。

我们可以创建一个绑定到我们Activity生命周期的[Job](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/index.html)实例，来管理我们的协程的生命周期。正如下面的示例代码一样，我们可以使用[Job()](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/invoke.html)工厂函数去创建一个Job对象的实例。我们需要确保全部的协程启动的时候，在它们各自的上下文中使用我们之前创建的Job实例，并且我们只需要在这个Job实例上调用一次[Job.cancel](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental/-job/cancel.html)函数，就能将它们全部停止。

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val job = Job() // create a job object to manage our lifecycle
    // now launch ten coroutines for a demo, each working for a different time
    val coroutines = List(10) { i ->
        // they are all children of our job object
        launch(context + job) { // we use the context of main runBlocking thread, but with our own job object 
            delay(i * 200L) // variable delay 0ms, 200ms, 400ms, ... etc
            println("Coroutine $i is done")
        }
    }
    println("Launched ${coroutines.size} coroutines")
    delay(500L) // delay for half a second
    println("Cancelling job!")
    job.cancel() // cancel our job.. !!!
    delay(1000L) // delay for more to see if our coroutines are still working
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-context-09.kt)获取完整的示例代码

这个实例的输出如下：

```java
Launched 10 coroutines
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Cancelling job!
```

我们可以看到，只有前3个协程输出了内容，并且我们仅仅调用了一次 `job.cancel()` 就将其余的协程取消了。所以，在刚刚我们假设的Android应用中，我们需要在Activity创建的时候去创建一个公共的父Job实例，并且将它应用到子协程中，当Activity销毁的时候，将这些子协程取消。


## 通道

Deferred 提供了一个在协程之间传递单个值的便利的方式，而通道则提供了一种传递流（Stream）的途径。

### 通道的基本用法

从概念上来说，[通道(Channel)](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-channel/index.html)与 `BlockingQueue` 非常相似。有一点不同的是，通道将阻塞的 `put` 函数替换为了[send](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-send-channel/send.html)挂起函数，将 `take` 函数替换成了 [receive](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-receive-channel/receive.html)挂起函数。

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch(CommonPool) {
        // this might be heavy CPU-consuming computation or async logic, we'll just send five squares
        for (x in 1..5) channel.send(x * x)
    }
    // here we print five received integers:
    repeat(5) { println(channel.receive()) }
    println("Done!")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-01.kt)获取完整的示例代码

y以上代码将会输出：

```java
1
4
9
16
25
Done!
```

### 通道的关闭和遍历

与队列不同，一个通道可以被关闭以明确的指明不会再有新的元素增加了。在数据接受者的一方，我们可以方便的使用 `for` 循环从通道中接收数据元素。

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch(CommonPool) {
        for (x in 1..5) channel.send(x * x)
        channel.close() // we're done sending
    }
    // here we print received values using `for` loop (until the channel is closed)
    for (y in channel) println(y)
    println("Done!")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-02.kt)获取完整的示例代码

### 构造通道的生产者

使用协程创建一个元素序列，是一个很普通的模式，我们经常可以在并发的代码中找到所谓的“生产者-消费者”模式。你可以将一个生产者抽象为一个函数，这个函数将通道作为结果返回，但是这种结果由函数作为结果进行返回的方式与通常的做法相反。

在生产者这端，你可以方便的使用一个名字叫做[produce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/produce.html)的协程构造器进行创建，在消费者这端，你可以使用该协程的拓展方法[consumeEach](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/consume-each.html)来代替传统的 `for` 循环。

```java
fun produceSquares() = produce<Int>(CommonPool) {
    for (x in 1..5) send(x * x)
}

fun main(args: Array<String>) = runBlocking<Unit> {
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-03.kt)获取完整的示例代码

### 管道

管道是当一个协程可能产生无限的数据流的时候使用的一种模式。

```java
fun produceNumbers() = produce<Int>(CommonPool) {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}
```

并且另外一个或多个协程将会消费流数据，做一些处理，并且会生成其他的结果，下边的示例仅仅是计算值的平方：

```java
fun square(numbers: ReceiveChannel<Int>) = produce<Int>(CommonPool) {
    for (x in numbers) send(x * x)
}
```

主函数开始执行并且连接整个管道：

```java
fun main(args: Array<String>) = runBlocking<Unit> {
    val numbers = produceNumbers() // produces integers from 1 and on
    val squares = square(numbers) // squares integers
    for (i in 1..5) println(squares.receive()) // print first five
    println("Done!") // we are done
    squares.cancel() // need to cancel these coroutines in a larger app
    numbers.cancel()
}
```

> 你可以在[这里](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/src/test/kotlin/guide/example-channel-04.kt)获取完整的示例代码

在这个示例中，我们不得不取消这些协程，因为之前介绍过，“协程类似于守护线程”，在一个大型应用中，如果我们不在需要使用它，我们需要停掉我们的管道。另一方面，我们可以将管道协程作为一个子协程进行使用。





























