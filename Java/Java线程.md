# 线程

## ThreadLocal 原理和应用

`ThreadLocal` 是 Java 提供的一种线程局部变量机制，它的核心目的是为每个线程提供独立的变量副本，使得不同线程对变量的操作相互隔离。以下深入剖析其实现原理：

### 1. 关键类与数据结构

- **`ThreadLocal` 类**：这是提供线程局部变量功能的核心类。它主要提供了 `get()`、`set(T value)`、`remove()` 以及 `initialValue()` 等方法，用于获取、设置、移除线程局部变量以及提供初始值。
- **`Thread` 类**：每个 `Thread` 对象内部都包含一个 `ThreadLocal.ThreadLocalMap` 类型的成员变量 `threadLocals`。这个 `ThreadLocalMap` 用于存储当前线程的所有 `ThreadLocal` 变量及其对应的值，是实现线程局部变量的关键数据结构。
- **`ThreadLocalMap` 类**：它是 `ThreadLocal` 的内部类，本质上是一个定制化的哈希表。`ThreadLocalMap` 使用开放地址法（线性探测）来解决哈希冲突，而不是像 `HashMap` 那样使用链表或红黑树。它的键是 `ThreadLocal` 实例的弱引用，值则是线程局部变量的值。

### 2. `set` 方法实现原理

当调用 `ThreadLocal` 的 `set(T value)` 方法时，执行以下步骤：

1. **获取当前线程**：首先通过 `Thread.currentThread()` 获取当前正在执行的线程对象。
2. **获取 `ThreadLocalMap`**：从当前线程对象中获取 `threadLocals` 成员变量，即 `ThreadLocalMap`。如果 `threadLocals` 为空，说明当前线程还没有为 `ThreadLocal` 变量分配存储空间，则调用 `createMap(Thread t, T firstValue)` 方法创建一个新的 `ThreadLocalMap`，并将当前 `ThreadLocal` 实例和传入的值作为键值对存入其中。`createMap` 方法内部会初始化一个默认大小为 16 的数组来存储键值对。
3. **设置值**：如果 `threadLocals` 不为空，则直接在 `ThreadLocalMap` 中设置值。`ThreadLocalMap` 使用 `ThreadLocal` 实例作为键，通过哈希算法计算出哈希值，再经过处理得到数组索引。如果该索引位置已经有元素（发生哈希冲突），则使用线性探测法寻找下一个可用的位置来存储键值对。

### 3. `get` 方法实现原理

调用 `ThreadLocal` 的 `get()` 方法时，执行以下操作：

1. **获取当前线程及 `ThreadLocalMap`**：同样先通过 `Thread.currentThread()` 获取当前线程，然后从线程对象中获取 `threadLocals`。
2. **获取值**：如果 `threadLocals` 不为空，`ThreadLocalMap` 会使用 `ThreadLocal` 实例作为键来查找对应的值。通过哈希算法计算哈希值并得到数组索引，然后从该索引位置开始查找。如果在该位置找到对应的键值对，则返回值；如果发生哈希冲突，会按照线性探测法继续查找下一个位置，直到找到匹配的键或者遇到空位置（说明该 `ThreadLocal` 变量在当前线程中未设置）。
3. **返回初始值**：如果 `threadLocals` 为空或者在 `ThreadLocalMap` 中没有找到对应的值，则调用 `initialValue()` 方法返回一个初始值。`initialValue()` 方法默认返回 `null`，通常需要子类重写该方法来提供自定义的初始值。

### 4. `remove` 方法实现原理

调用 `ThreadLocal` 的 `remove()` 方法时：

1. **获取当前线程及 `ThreadLocalMap`**：先获取当前线程及其 `threadLocals`。
2. **移除键值对**：如果 `threadLocals` 不为空，`ThreadLocalMap` 会使用 `ThreadLocal` 实例作为键来移除对应的键值对。找到对应的键值对后，将该位置的元素设为 `null`，并通过线性探测法对后续元素进行调整，以确保哈希表的一致性。这一步操作很重要，因为 `ThreadLocalMap` 使用弱引用存储键，如果不及时移除不再使用的键值对，可能会导致内存泄漏。

### 5. 弱引用与内存管理

`ThreadLocalMap` 使用弱引用（`WeakReference`）来存储 `ThreadLocal` 实例作为键。这意味着当 `ThreadLocal` 对象在其他地方没有强引用指向它时，在垃圾回收时，`ThreadLocal` 实例会被回收。然而，如果 `Thread` 对象一直存活（例如线程池中的线程），并且 `ThreadLocalMap` 中存在对 `ThreadLocal` 实例的弱引用，即使 `ThreadLocal` 对象在外部已不可达，`ThreadLocalMap` 中的键值对也不会被自动清理。如果后续不再使用该 `ThreadLocal` 变量，就需要手动调用 `remove()` 方法从 `ThreadLocalMap` 中移除对应的键值对，以避免内存泄漏。

综上所述，`ThreadLocal` 通过 `Thread` 类中的 `ThreadLocalMap` 实现了线程局部变量的存储与管理，通过巧妙的设计确保了不同线程之间变量的隔离性，同时在使用过程中需要注意内存管理以避免潜在的内存泄漏问题。

### 6. 在复杂业务逻辑中的应用

模拟的银行转账操作，每个线程处理一笔转账，并且每个线程需要记录自己的操作日志。

```java
class TransferLog {
    private final String threadName;
    private final Date timestamp;
    private final String operation;

    public TransferLog(String threadName, Date timestamp, String operation) {
        this.threadName = threadName;
        this.timestamp = timestamp;
        this.operation = operation;
    }

    @NonNull
    @Override
    public String toString() {
        return "Thread: " + threadName + ", Time: " + timestamp + ", Operation: " + operation;
    }
}

public class BankTransfer {
    private static final ThreadLocal<TransferLog> threadLocalLog = ThreadLocal.withInitial(() -> null);

    public static void transfer(double amount, String fromAccount, String toAccount  {
    String record = "Transfer $" + amount + ", from " + fromAccount + " to " + toAccount;
    System.out.println(record);
    TransferLog log = new TransferLog(Thread.currentThread().getName(), new Date(), record);
    threadLocalLog.set(log);
}

    public void printLog() {
        TransferLog log = threadLocalLog.get();
        if (log != null) {
            System.out.println(log);
        }
            threadLocalLog.remove();
        }
}

@Test
public void testThreadMethod1() {
    int num = 2;
    for (int i = 0; i < num; i++) {
        int finalI = i;
        Thread t = new Thread(() -> {
            try {
                BankTransfer.transfer(100.0 + finalI * 100, "A_" + finalI, "B_" + finalI);
                BankTransfer.printLog();
                Thread.sleep(100);
            } catch (InterruptedException e) {
                    throw new RuntimeException(e);
            }
        });
        t.start();
        try {
            t.join();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
    System.out.println("Both threads have completed.");
}
```

## ExecutorService 原理和应用

### 原理讲解

1. **线程池概念**：`ExecutorService` 通常基于线程池来实现。线程池是一个预先创建好的线程集合，这些线程可以被复用执行提交的任务。线程池的使用避免了频繁创建和销毁线程带来的开销，提高了系统性能和资源利用率。
2. **核心组件**：
   - **线程池管理器**：负责创建、管理和维护线程池中的线程。它控制线程的数量，根据任务的提交情况动态调整线程池的大小。
   - **工作队列**：用于存放等待执行的任务。当线程池中的线程都在忙碌时，新提交的任务会被放入工作队列中排队等待执行。工作队列的类型会影响线程池的行为，常见的有 `ArrayBlockingQueue`（有界队列）、`LinkedBlockingQueue`（无界队列）、`SynchronousQueue`（同步队列）等。
   - **线程池中的线程**：这些线程从工作队列中获取任务并执行。线程在执行完一个任务后，不会被销毁，而是回到线程池中等待执行下一个任务，从而实现线程的复用。
3. **线程池状态**：`ThreadPoolExecutor`（`ExecutorService` 的一个常用实现类）使用一个 `int` 类型的变量 `ctl` 来同时表示线程池的状态（`runState`）和线程池中有效线程的数量（`workerCount`）。通过位运算来区分这两个信息。线程池有以下几种状态：
   - **`RUNNING`**：线程池正常运行，可以接受新任务并处理队列中的任务。
   - **`SHUTDOWN`**：不再接受新任务，但会继续处理队列中已有的任务。
   - **`STOP`**：不再接受新任务，停止处理队列中的任务，并中断正在执行的任务。
   - **`TIDYING`**：所有任务都已终止，`workerCount` 为 0，线程池即将进入 `TERMINATED` 状态。
   - **`TERMINATED`**：线程池已完全终止。
4. **任务提交与执行流程**：
   - **提交任务**：当调用 `execute(Runnable task)` 方法提交一个任务时，`ThreadPoolExecutor` 首先会检查线程池的状态。如果线程池处于 `RUNNING` 状态，会尝试将任务添加到工作队列中。
   - **线程创建与任务分配**：
     - 如果任务成功添加到工作队列，`ThreadPoolExecutor` 会再次检查线程池状态，确保线程池仍然是 `RUNNING` 状态（因为在添加任务到队列的过程中，线程池状态可能发生变化）。如果是 `RUNNING` 状态，且线程池中的活跃线程数为 0 或者小于核心线程数（`corePoolSize`），会创建一个新的线程来执行任务队列中的任务。
     - 如果任务无法添加到工作队列（队列已满），且当前线程数小于最大线程数（`maximumPoolSize`），会创建一个新的线程来执行该任务。
   - **拒绝任务**：如果任务无法添加到工作队列且当前线程数已达到最大线程数，`ThreadPoolExecutor` 会根据设置的拒绝策略来处理该任务，例如抛出异常（`AbortPolicy`，默认策略）、由调用者线程执行任务（`CallerRunsPolicy`）、丢弃任务（`DiscardPolicy`）或丢弃队列中最老的任务（`DiscardOldestPolicy`）。
   - **线程执行任务**：线程池中的线程从工作队列中取出任务并执行。当一个线程执行完一个任务后，它会再次从工作队列中获取新的任务继续执行，直到任务队列为空或者线程池状态发生变化导致线程终止。
5. **线程生命周期管理**：
   - **核心线程**：核心线程在创建后默认情况下不会被销毁，即使它们处于空闲状态，除非设置了 `allowCoreThreadTimeOut(true)`，此时核心线程在空闲时间超过 `keepAliveTime` 后也会被销毁。
   - **非核心线程**：非核心线程在空闲时间超过 `keepAliveTime` 后会被销毁，以减少资源消耗。这有助于线程池根据任务负载动态调整线程数量，提高资源利用率。
6. **关闭线程池**：
   - **`shutdown()`**：调用此方法后，线程池进入 `SHUTDOWN` 状态，不再接受新任务，但会继续处理队列中已有的任务。线程池中的线程会在执行完当前任务和队列中的任务后逐渐终止。
   - **`shutdownNow()`**：调用此方法后，线程池进入 `STOP` 状态，不再接受新任务，停止处理队列中的任务，并尝试中断正在执行的任务。它会返回等待在任务队列中尚未执行的任务列表。

### 使用流程

1. **创建 `ExecutorService` 实例**：可以通过 `Executors` 类的静态方法创建不同类型的 `ExecutorService` 实例，也可以直接创建 `ThreadPoolExecutor` 实例来自定义线程池的参数。
   
   **使用 `Executors` 创建线程池**：
   
   - **固定大小线程池**：`ExecutorService executor = Executors.newFixedThreadPool(int nThreads)`，创建一个固定大小的线程池，其中 `nThreads` 是线程池中线程的数量。适用于已知并发任务数量且任务执行时间较短的场景。
   - **缓存线程池**：`ExecutorService executor = Executors.newCachedThreadPool()`，创建一个可缓存的线程池。如果线程池中的线程空闲时间超过 60 秒，会被销毁。适用于任务执行时间较短但数量不定的场景。
   - **单线程线程池**：`ExecutorService executor = Executors.newSingleThreadExecutor()`，创建一个只有一个线程的线程池。所有任务按照提交顺序依次执行，适用于需要顺序执行任务的场景。
   
   **直接创建 `ThreadPoolExecutor`**：  
   
   ```java
   int corePoolSize = 5;
   int maximumPoolSize = 10;
   long keepAliveTime = 10;
   TimeUnit unit = TimeUnit.SECONDS;
   BlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(10);
   RejectedExecutionHandler handler = new ThreadPoolExecutor.AbortPolicy();
   ExecutorService executor = new ThreadPoolExecutor(
       corePoolSize,
       maximumPoolSize,
       keepAliveTime,
       unit,
       workQueue,
       handler);
   ```
   
   - `corePoolSize` 是核心线程数，即线程池中始终保持活动的线程数量。
   - `maximumPoolSize` 是线程池允许的最大线程数。
   - `keepAliveTime` 和 `unit` 定义了非核心线程在空闲状态下的存活时间。
   - `workQueue` 是任务队列，用于存放等待执行的任务。
     - `handler` 是拒绝策略，当任务无法添加到队列且线程数达到最大线程数时，用于处理新提交的任务。

2. **提交任务**：
   
   - 使用 `execute(Runnable task)` 方法提交一个 `Runnable` 任务。`Runnable` 是一个接口，只包含一个 `run` 方法，没有返回值。例如：
     
     ```java
     executor.execute(() -> {
           System.out.println("Task is running.");
     });
     ```
     
     - 使用 `submit(Callable<T> task)` 方法提交一个 `Callable` 任务。`Callable` 也是一个接口，其中的 `call` 方法可以返回一个值，并且可以抛出异常。例如：
     
     ```java
     Callable<Integer> callableTask = () -> {
        return 1;
     };
     Future<Integer> future = executor.submit(callableTask);
     try {
        Integer result = future.get();
        System.out.println("Task result: " + result);
     } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();
     }
     ```
     
         在上述代码中，`submit` 方法返回一个 `Future` 对象，可以通过 `Future` 对象的 `get` 方法获取任务的执行结果。`get` 方法会阻塞当前线程，直到任务执行完毕并返回结果。  
   
   3. **关闭 `ExecutorService`**：
       在使用完 `ExecutorService` 后，需要关闭它以释放资源。调用 `shutdown()` 方法，线程池会停止接受新任务，但会继续处理队列中已有的任务。例如：
   
   ```java
   executor.shutdown();
    try {
       if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
           executor.shutdownNow();
           if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
              System.err.println("Pool did not terminate");
           }
       }
    } catch (InterruptedException ie) {
       executor.shutdownNow();
       Thread.currentThread().interrupt();
    }
   ```
- 在上述代码中，调用 `shutdown()` 方法后，使用 `awaitTermination` 方法等待线程池中的任务执行完毕，等待时间为 60 秒。如果在 60 秒内任务没有执行完，调用 `shutdownNow()` 方法尝试停止任务，并再次等待 60 秒。

- 调用 `shutdownNow()` 方法，线程池会立即停止接受新任务，停止处理队列中的任务，并尝试中断正在执行的任务，返回等待在任务队列中尚未执行的任务列表。
