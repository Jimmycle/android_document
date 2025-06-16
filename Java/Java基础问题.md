# Java 基础问题

## HashMap

##### Q1: 阐述 JDK8 中 HashMap 基础概念

A：

1. 存储结构：在 JDK8 中HashMap采用 “数组 + 链表 + 红黑树” 的存储结构。它有一个数组（`Node[] table`），数组中的每个元素被称为桶（bucket）。当往 HashMap 中 put 键值对时，通过对键的哈希值进行计算，确定该键值对应该存放在数组的哪个bucket中。如果桶bucket中没有元素，就直接放入；如果bucket中已有元素（即发生哈希冲突），则以链表或红黑树的形式存储在该桶位置。

2. Node 节点：基本存储单元是 `Node` 类（在 JDK8 中）。`Node` 类实现了 `Map.Entry` 接口，包含了键（`key`）、值（`value`）、下一个节点引用（`next`）以及哈希值（`hash`）等属性。

3. 哈希函数：为了将键值对均匀地分布在数组中，HashMap 使用了哈希函数。首先通过 `key.hashCode()` 方法获取键的哈希码，然后经过一系列位运算（如 `hashCode()` 与自身右移 16 位的结果进行异或运算等），得到最终用于确定数组索引的哈希值。这样做的目的是为了让哈希值散列更平均，减少哈希冲突。

##### Q2: JDK8 中 HashMap 如何扩容

A：

1. 扩容条件：HashMap 有两个重要参数影响扩容，即负载因子（`loadFactor`）和当前容量（`capacity`）。负载因子默认值为 0.75，当 HashMap 中已存储的键值对数量（`size`）超过 `capacity * loadFactor` 时，就会触发扩容。例如，初始容量为 16，负载因子为 0.75，那么当 `size` 达到 `16 * 0.75 = 12` 时，就会进行扩容。
2. 扩容过程：
   - 创建新数组：扩容时，会创建一个新的数组，新数组的容量是原数组容量的两倍。例如，原数组容量为 16，扩容后新数组容量为 32。
   - 重新分配元素：JDK8 之前，需要对每个元素重新计算哈希值并重新插入到新数组位置。但在 JDK8 中，对于链表或者红黑树存储的元素，会根据元素的哈希值与旧数组容量的关系，决定是留在原位置还是移动到新位置 + 旧数组容量的位置。具体来说，如果元素的哈希值与旧数组容量进行与运算（`hash & oldCap`）结果为 0，则留在原位置；否则移动到新位置 + 旧数组容量的位置。这个过程减少了重新计算哈希值的次数，提高了扩容效率。这样的扩容机制，HashMap 能够动态调整自身容量，以适应不断增加的数据量，同时保持良好的性能。

##### Q3：JDK8 中 HashMap 为什么要引入红黑树？

A：引入红黑树提升查找性能，当哈希冲突严重时，多个元素会被存储在同一个的链表中，查找时间复杂度会退化为 O (n)。红黑树作为一种自平衡二叉搜索树，其查找、插入和删除操作平均时间复杂度为 O (log n)，在链表长度较长时，将链表转换为红黑树能显著提升查找性能。

##### Q4：JDK8 中 HashMap 链表转红黑树的条件哪些？

A：

1. 链表长度达到 8，当同一个bucket中的链表长度达到 8 时，会触发链表转红黑树的条件判断。这是基于概率统计得出的一个经验值。

2. 当数组容量达到 64，仅仅链表长度达到 8 还不够，还需要数组容量达到 64 才会将链表转换为红黑树。因为在数组容量较小时，哈希冲突相对容易通过扩容来解决，而不需要直接转换为红黑树。只有当数组容量足够大，且链表长度达到 8 时，转换为红黑树才是更优的选择。

## 泛型

##### Q1: Java 的泛型有什么作用，使用需要注意哪些？

A：

1. 类型安全：泛型允许在编译时指定数据类型，从而避免在运行时出现类型转换异常。

2. 代码复用：泛型使得可以编写通用的代码，适用于多种数据类型，而无需为每种数据类型都编写重复的代码。例如，`Collections` 类中的 `sort` 方法，它可以对实现了 `Comparable` 接口的任何类型的列表进行排序

3. 由于类型擦除，在运行时泛型类型信息被擦除，通过反射获取泛型类型信息可能无法得到预期结果。

4. 导致泛型方法在重载时出现意外情况，因为擦除后方法签名可能变得相同。比如尝试定义两个仅泛型参数不同的重载方法 `print(T t)` 和 `print(List<T> list)`，编译时会报错。因为在类型擦除后，这两个方法的签名变为相同的 `print(Object)`，Java 不允许这种形式的方法重载。

5. 由于类型擦除，不能直接创建泛型数组，因为运行时无法确定数组元素的实际类型。比如 List[] stringLists = new ArrayList[10] 不允许

## Lambda表达式

##### Q1: Java中的Lambda表达式和方法引用如何实现延迟执行？

A：Lambda 表达式本质上是一个匿名函数，它用一种简洁的方式来表示可传递给方法或存储在变量中的代码块。Lambda 表达式不会在定义时立即执行，而是在其被调用时执行。例如：

```java
Runnable runnable = () -> System.out.println("Lambda 表达式");
// 这里仅仅是定义了一个 Runnable 对象，Lambda 表达式中的代码并未执行
runnable.run(); 
// 调用 run 方法时，Lambda 表达式中的代码才会执行，实现了延迟执行
```

方法引用是一种更简洁地引用已存在方法的方式，同样也实现了延迟执行。它可以看作是 Lambda 表达式的一种特殊形式，当 Lambda 表达式只调用一个已存在的方法时，可以使用方法引用。例如：

```java
Consumer<String> consumer = System.out::println;  
consumer.accept("test println");
```

##### Q2：在Android开发中，Lambda表达式和方法引用应用场景？

A：在 Android 开发中，

1. 为各种视图组件设置事件监听器。使用 Lambda 表达式和方法引用可以使代码更加简洁。

2. 使用异步任务来执行一些耗时操作，避免阻塞主线程。例如 Handler 进行异步操作。

3. 集合数据的过滤。stream, fliter, collect

4. RxJava响应式编程库

## Java8 函数式编程相关

##### Q1：什么是函数式接口?

A：函数式接口是指只包含一个抽象方法的接口。Java 8 引入了 `@FunctionalInterface` 注解来标识函数式接口，虽然使用该注解不是强制的，但加上它可以让编译器进行更严格的检查，确保接口确实是函数式接口。例如：

```java
@FunctionalInterface
interface MyFunctionalInterface {
    void execute();
}
MyFunctionalInterface myFunctionalInterface = () -> System.out.println("Lambda 实现函数式接口");
myFunctionalInterface.execute();
```

##### Q2：什么是 Stream API？

A：Stream API 是 Java 8 引入的用于处理集合数据的新特性。它允许以一种声明式、函数式的方式对集合进行过滤、映射、等操作。API 提供了丰富的操作符，分为中间操作（如 `filter`、`map` 等）和 终端操作（如 `forEach`、`collect` 等）。中间操作返回一个新的 Stream，可以链式调用多个中间操作；终端操作会触发 Stream 的计算，返回一个最终结果或副作用。支持并行处理，Stream API 支持并行处理集合数据，利用多核 CPU 的优势提高处理速度。只需将 `stream()` 方法替换为 `parallelStream()` 即可

##### Q3：Optional 类的作用？

A：避免空指针异常：`Optional` 类是 Java 8 引入的，主要用于解决空指针异常（`NullPointerException`）问题。在传统的 Java 编程中，当调用一个可能为 `null` 的对象的方法或访问其属性时，如果对象为 `null`，就会抛出 `NullPointerException`。`Optional` 类提供了一种更安全的方式来处理可能为 `null` 的值，通过将值包装在 `Optional` 对象中，可以明确地表示值存在或不存在，从而在访问值之前进行必要的检查，避免空指针异常。它通过一些方法，如 `isPresent()`、`ifPresent(Consumer<? super T> action)`、`orElse(T other)` 等，让开发者能够以一种更可读的方式处理可能的空值情况，使代码的意图更加明确。

```java
public class TestActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);

        TextView textView = findViewById(R.id.text_view);
        Intent intent = getIntent();

        Optional<String> dataOptional = Optional.ofNullable(intent.getStringExtra("key"));
        dataOptional.ifPresent(data -> textView.setText(data));
        dataOptional.orElse("默认值");
    }
}
```
