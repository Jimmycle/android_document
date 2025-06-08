# Coroutine

## 一、协程和线程

线程是操作系统级别的概念，在操作系统中线程是抢占式的。

协程是编程语言级别的概念，在编程语言中协程是协作式的。

**但是，就我本人常用的语言Java来说，它里面线程也是协作式的。**



## 二、协程的基本概念

协程可以让代码执行时**挂起**，并在适当的时候**恢复**。它可以被看作是一个轻量级的线程。它允许我们在不阻塞线程的情况下执行异步操作。



## 三、协程的分类

### 3.1 按照调用栈分类

- 有栈协程：每个协程都会分配专门的调用栈，和类似我们JVM中线程的栈；可以在任意函数嵌套中挂起
- 无栈协程：不会分配单独的调用栈，挂起点状态通过闭包或对象保存；只能在当前函数中挂起

**Kotlin中的协程通过Continuation来保持挂起点的状态，所以Kotlin是无栈协程。**



### 3.2 按照调用关系分类

- 对称协程：调度权可以转移给任意的协程，协程之间的关系是对等的
- 非对称协程：调度权只能转移给调用自己的协程，协程之间存在父子关系

**Kotlin中的协程是非对称协程**



## 四、Kotlin协程的基本要素

### 4.1 挂起函数和Suspend关键字

- 挂起函数：以Suspend关键字修饰。

- 挂起函数只能在在其他挂起函数或协程中执行。



挂起函数执行时就包含了**挂起**的语义；而挂起函数返回时，包含了**恢复**的语义。

再次加深印象，协程的**核心思想就是挂起/恢复**



### 4.2 Continuation

#### 基本概念

`Continuation` 是 Kotlin 协程的基础接口之一，它是用来表示挂起函数执行到某个点后的剩余工作。具体来说，当一个挂起函数被调用并且达到挂起点（比如调用了另一个挂起函数），当前的执行上下文会被保存到一个 `Continuation` 实例中。然后，控制权返回给调用者，允许其他代码运行或者直接返回给事件循环。一旦挂起的原因解决了（例如网络请求完成了），之前保存的 `Continuation` 会被恢复，从而让挂起函数从暂停的地方继续执行下去。

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    
    public fun resumeWith(result: Result<T>)
}
```



挂起函数类型

```kotlin
suspend ()->Unit
suspend (Int)->String
```



似乎和Continuation没啥关系？

挂起函数中并没有出现Continuation！



其实挂起函数在Java中的样子是类似这样的。对应上面定义的两个挂起函数

```kotlin
(continuation:Continuation<Unit>)->Any

(Int, conitnuation:Continuation<String>)->Any
```



这个continuation对象不是我们传入的，而是Kotlin编译器给我们传入的！

它是挂起函数的最后一个参数。



#### 迟来的答案

之前提到的，挂起函数只能在挂起函数和协程中调用，为什么呢？

因为它需要一个continuation作为参数，编译只是帮我们传入参数，无法给我们无中生有生成一个continuation对象！

而这个continuation只有挂起函数和协程才有。



#### Continuation的泛型参数

Continuation有一个泛型参数，这个泛型参数类型是挂起函数本身的返回值决定的。



#### 返回Any的作用

- 当挂起函数没有真正挂起时，Any用来承载返回值真正的结果
- 当挂起函数挂起时，Any返回的是一个挂起标志COROUTINE_SUSPENDED，让协程体外面的调用者知道我们挂起了。



#### 如何拿到Continuation？

可以通过调用 `suspendCoroutine` 或者 `suspendCancellableCoroutine` 来实现这一点

```kotlin
import kotlin.coroutines.suspendCoroutine

suspend fun fetchData(): String {
    return suspendCoroutine { continuation: Continuation<String> ->
        // 模拟异步操作
        doAsyncOperation {
            val result = "Data"
            // 当异步操作完成时，恢复协程
            continuation.resumeWith(Result.success(result))
        }
    }
}
```

在这个例子中，`suspendCoroutine` 捕获了当前的 `Continuation<String>` 并允许你在异步操作完成后通过调用 `continuation.resumeWith(...)` 来恢复协程。



#### Continuation什么时候真正的挂起？

当发起真正的异步调用时。也就是常说的切线程。

上面例子就是属于真正挂起了。



### 4.3 协程的创建

上面已经提到了协程需要Continuation，那么自然而然的就会带来一个问题，第一个Continuation哪里来的？



创建协程的API：

**createCoroutine**

```kotlin
public fun <T> (suspend () -> T).createCoroutine(
    completion: Continuation<T>
): Continuation<Unit>

public fun <R, T> (suspend R.() -> T).createCoroutine(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit>
```



createCoroutine会返回一个Continuation的对象，这个对象其实是给我们手动启动协程的。

调用他的resume函数，协程就启动了。他是协程的本体。

协程执行完的时候会resume作用参数的completion，这个也是一个Continuation



但这样每次创建完都resume一下，显得不太聪明的样子！

所以还有一个创建完直接启动的API

**startCoroutine**



#### 创建的这个协程会resume几次呢？

首先，被创建出来Continuation需要被resume一下才能执行，然后completion这个Continuation也会resume一下，最后suspend function本身也是一个Continuation，也会resume一次。

所以是3次。



如何里面N个真正的挂起点呢，那就是N+2次。

每个挂起点resume一次，外加启动协程时resume协程的本体，还有结束是completion resume一下。



### 4.3 CoroutineContext

CoroutineContext是一个数据集合，是数据的载体，用来包装数据的。

CoroutineContext.Key

CoroutineContext.Element

看着就很像一个我们的熟悉的MAP。



这个东西是可以自定义的。当然官方已经提供了很多特别好用的上下文了。

- **Job**: 表示协程的工作单元，负责管理协程的生命周期。它的键是 `Job.Key`。
- **CoroutineDispatcher**: 决定了协程运行所在的线程或线程池。它的键是 `ContinuationInterceptor.Key`（注意：尽管 `CoroutineDispatcher` 实现了 `ContinuationInterceptor` 接口，但在实践中，它通常被视为调度器）。
- **CoroutineName**: 给协程命名，便于调试。它的键是 `CoroutineName.Key`。
- **CoroutineExceptionHandler**: 指定当协程内部发生未捕获异常时的处理方式。它的键是 `CoroutineExceptionHandler.Key`。

#### ContinuationInterceptor

这是一个比较特殊的接口`ContinuationInterceptor` 

```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>

    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
}
```

这个玩意儿很好玩啊，输入一个Continuation，还给你一个Continuation。

真狸猫换太子！

这里可以操作空间就很大了。负责切换线程的**CoroutineDispatcher**就是基于这个实现的。

### 4.4 SuspendLambda

`SuspendLambda` 是 Kotlin 编译器内部使用的一个接口，它并不是直接供开发者使用的 API。实际上，当你定义一个 `suspend` lambda 或者任何挂起函数时，Kotlin 编译器会在后台生成实现 `SuspendLambda` 接口的类。这个接口是 Kotlin 用来支持协程和挂起函数调用机制的一部分。

#### 深入理解 SuspendLambda

`SuspendLambda` 实际上是 Kotlin 标准库中 `FunctionN`（如 `Function0`, `Function1`, 等）接口的一种特殊形式，专门用于挂起函数或 lambda 表达式。它帮助 Kotlin 协程机制管理协程的状态机以及挂起点。每个挂起 lambda 在编译时都会被转换成实现了 `SuspendLambda` 接口的匿名类实例。

#### 主要特性

- **状态机**: 当你有一个包含多个挂起点的挂起 lambda 或函数时，Kotlin 编译器会自动生成一个状态机来跟踪该函数/lambda 的执行进度。
- **Continuation 参数**: 每个挂起函数在编译后都会接收一个额外的 `Continuation` 参数，这是为了能够在挂起点之后恢复执行流程。`SuspendLambda` 同样涉及到与 `Continuation` 的交互以实现这一点。



### 4.5 SafeContinuation

`SafeContinuation` 是 Kotlin 协程库内部使用的一个辅助类，它帮助确保 `Continuation` 的调用是安全的，尤其是避免重复调用 `resume` 或 `resumeWithException` 方法，这可能会导致未定义行为或崩溃。虽然 `SafeContinuation` 主要是一个实现细节，并不是直接供开发者使用的 API，但了解它的存在和作用有助于深入理解 Kotlin 协程的工作机制。

#### SafeContinuation 的主要作用

1. **防止重复恢复**: 在协程中，一旦 `Continuation` 被恢复（通过调用 `resume` 或 `resumeWithException`），理论上不应该再次尝试恢复它。然而，在复杂的异步代码中，由于错误处理不当或其他原因，可能会意外地多次调用这些方法。`SafeContinuation` 提供了一层保护，以确保即使发生这种情况也不会导致程序崩溃。
2. **简化异常处理**: 它还可以简化异常处理逻辑，使得在不同情况下都能正确地传播异常，而不必担心异常丢失或被不恰当地处理。

#### 工作原理

`SafeContinuation` 通常作为一个包装器，包裹原始的 `Continuation` 实例，并在其上提供额外的安全检查。当你通过 `SafeContinuation` 来恢复一个协程时，它会首先检查该协程是否已经被恢复过：

- 如果协程尚未被恢复，则正常进行恢复操作。
- 如果协程已经被恢复，则忽略后续的恢复请求，或者根据具体实现采取其他措施（如抛出异常）。



### 4.6 拦截器的调用位置

拦截器实际的调用时机在SafeContinuation 和 `SuspendLambda`之间，既保证了安全性，又会在实际的`SuspendLambda`之前调用。在`SuspendLambda`之前调用当然就能决定`SuspendLambda`跑在什么线程了



