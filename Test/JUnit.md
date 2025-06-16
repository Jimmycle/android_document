# JUnit单元测试

在 JUnit 单元测试中显示多线程内容可能会面临一些挑战，因为多线程的异步特性可能导致测试结果不稳定。可以通过以下几种方式在 JUnit 测试中处理和显示多线程相关的内容：

### 1. 使用`CountDownLatch`同步线程

`CountDownLatch` 可以用来同步多个线程，确保在所有线程完成操作后再进行断言。

```java
public class MultithreadedTest {
    @Test
    public void testMultithreadedOperation() throws InterruptedException {
        int numThreads = 3;
        CountDownLatch latch = new CountDownLatch(numThreads);
        ExecutorService executorService = Executors.newFixedThreadPool(numThreads);
        for (int i = 0; i < numThreads; i++) {
            executorService.submit(() -> {
                try {
                    // 模拟多线程执行的任务
                    System.out.println(Thread.currentThread().getName() + " is running.");
                    // 模拟一些工作
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    latch.countDown();
                }
            });
        }
        latch.await();
        executorService.shutdown();
        System.out.println("All threads have completed.");
    }
}
```

在上述代码中：

- `CountDownLatch` 的构造函数接受一个整数参数，表示需要等待的线程数量。
- 每个线程在完成任务后调用 `latch.countDown()` 方法，使计数减一。
- `latch.await()` 方法会阻塞当前线程，直到 `CountDownLatch` 的计数为零，即所有线程都完成了任务。

###### 2. 使用`Thread.join()`方法

`Thread.join()` 方法可以使当前线程等待调用该方法的线程执行完毕。

```java
public class MultithreadedTestWithJoin {
    @Test
    public void testMultithreadedWithJoin() throws InterruptedException {
        Thread thread1 = new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " is running.");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        Thread thread2 = new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " is running.");
            try {
                Thread.sleep(1500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        thread1.start();
        thread2.start();

        thread1.join();
        thread2.join();
        System.out.println("Both threads have completed.");
    }
}
```

这里：

- 启动两个线程 `thread1` 和 `thread2`。
- 主线程调用 `thread1.join()` 和 `thread2.join()`，这会使主线程等待 `thread1` 和 `thread2` 执行完毕后再继续执行，从而保证可以在所有线程完成后输出 “Both threads have completed.”，展示多线程的执行过程。

### 3. 使用`Phaser`进行多线程同步

`Phaser` 是 Java 7 引入的一个灵活的同步工具，可以用于控制多个线程在特定阶段的同步。

```java
import java.util.concurrent.Phaser;

public class MultithreadedTestWithPhaser {
    @Test
    public void testMultithreadedWithPhaser() {
        int numThreads = 3;
        Phaser phaser = new Phaser(numThreads);
        phaser.register();
        for (int i = 0; i < numThreads; i++) {
            new Thread(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + " has registered.");
                    // 模拟工作
                    Thread.sleep(1000);
                    phaser.arriveAndAwaitAdvance();
                    System.out.println(Thread.currentThread().getName() + " has advanced.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
        // 主线程等待所有线程完成第一阶段
        phaser.arriveAndAwaitAdvance();
        System.out.println("All threads have reached the first phase.");
    }
}
```

在这个例子中：

- `Phaser` 的构造函数接受参与同步的线程数量。
- 每个线程通过 `phaser.register()` 方法注册到 `Phaser` 中，完成工作后调用 `phaser.arriveAndAwaitAdvance()`，表示已到达某个阶段并等待其他线程到达同一阶段。
- 主线程通过 `phaser.arriveAndAwaitAdvance()` 等待所有线程完成第一阶段，然后输出 “All threads have reached the first phase.”，展示多线程在特定阶段的同步情况。
