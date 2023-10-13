---
title: CompletableFuture-一种新的异步任务编排解决方案
date: 2023/10/13 15:03:48
categories:
- [JAVA, 多线程]
tags:
- Thread
- 多线程
---

### 前情提要

多线程主要是通过提高对 `CPU` 的利用率从而提升多任务处理的效率，以满足用户的友好体验。

根据 `Oracle` 官方文档的描述：`JAVA` 中创建线程的方式有两种：

1. 继承 `java.lang.Thread` 类。（ `Thread` 类本质上也是实现了 `Runnable` 接口）
2. 实现 `java.lang.Runnable` 接口。

但是，这两种方法都存在一个缺陷——没有返回值，也就是说我们无法得知线程执行结果。



### Future模型

为了解决这个问题，`JDK1.5` 引入了 `Future` 和 `Callable` 接口，我们只需要将创建的任务 `submit` 到线程池，之后我们可以执行其他业务逻辑，根据需要再通过 `java.util.concurrent.Future#get()` 方法就能够得到任务执行的返回值，节约程序的运行时间。

通过一个例子快速了解 `Future` 模型：

```java
public static void main(String[] args) throws Exception {
    // 为了方便测试，这里直接使用Executors创建线程池
    // 实际项目中，不推荐使用Executors创建线程池，其内部使用的是无界队列，容易造成OOM
    ExecutorService executorService = Executors.newFixedThreadPool(10);
    long start = System.currentTimeMillis();
    
    Future<String> future1 = executorService.submit(() -> {
        // 模拟任务1执行耗时
        Thread.sleep(300);
        return "任务1执行完成";
    });
    
    Future<String> future2 = executorService.submit(() -> {
        // 模拟任务2执行耗时
        Thread.sleep(500);
        return "任务2执行完成";
    });
    
    // 模拟主线程处理其他业务耗时
    Thread.sleep(300);
    
    // 获取任务1执行结果
    String result1 = future1.get();
    System.out.println(result1);
    
    // 获取任务2执行结果
    String result2 = future2.get();
    System.out.println(result2);
    
    long end = System.currentTimeMillis();
    System.out.println("总耗时：" + (end - start) + "ms");
    
    executorService.shutdown();
}
```

执行结果：

```tex
任务1执行完成
任务2执行完成
总耗时：564ms
```

可以看到，合理使用异步任务可以大大提高程序的执行效率。



### Future的局限性

从上面实例的输出我们可以看到，调用 `java.util.concurrent.Future#get()` 方法会一直阻塞主线程。

简单业务上，我们可以使用 `Future` 的另一个重载方法 `get(long, TimeUnit)` 来设置超时时间，避免主线程被永远阻塞。

`get()` 方法原文描述：

> Waits if necessary for the computation to complete, and then retrieves its result.

当然 `Future` 还贴心的提供了一个 `isDone()` 方法，可以在程序中**轮询**调用这个方法，等待处理完成后再调用 `get()` 方法获取返回值。

注意！如果任务完成，调用 `isDone()` 方法将返回 `true` ，**但完成可能是由于正常终止、异常或取消——在所有这些情况下，此方法都将返回true**。

而对于 `JDK8` 中的 `isDone()` 方法还存在这样一个 `BUG`，目前已在 `JDK9` 修复，大家感兴趣可以去看看。[JDK-8073704](https://bugs.openjdk.org/browse/JDK-8073704)

`isDone()` 方法原文描述：

> Returns true if this task completed. Completion may be due to normal termination, an exception, or cancellation -- in all of these cases, this method will return true.

但无论是哪种方法，`Future` 对于处理结果的获取似乎都显得不够友好。

**阻塞的方式和异步编程的设计理念相违背，而轮询的方式又会无谓的耗费CPU资源**。

那么~ 就只能到此为止了吗？

当然不是，`Doug Lea` 大神在 `JDK8` 又给我们带来了并发编程神器—— `CompletableFuture` 。



### CompletableFuture

`JAVA8` 引入的 `CompletableFuture`  解决了 `Future` 在实际使用过程中不支持异步任务的编排组合以及阻塞获取任务返回值的问题。

除了提供更为好用的 `Future` 特性之外，`CompletableFuture` 还提供了函数式编程、异步任务编排组合（可以将多个异步任务串联起来，组成一个完整的链式调用）等能力。

`CompletableFuture` 类结构示意图：

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {}
```



![image-20231013170446028](https://blog.seeyourface.cn/blog/image-20231013170446028.png)

`CompletableFuture` 同时实现了 `Future` 和 `CompletionStage` 接口:

`CompletionStage` 接口描述了一个异步计算的阶段。多个计算可以分成多个阶段或步骤，此时可以通过它将所有步骤组合起来，形成异步计算的流水线。

`CompletableFuture` 通过继承 `CompletionStage` 获取其提供的函数式能力。从这个接口的方法参数可以发现其大量使用了 `Java8` 引入的函数式编程。

![image-20231013171231135](https://blog.seeyourface.cn/blog/image-20231013171231135.png)



### CompletableFuture使用

#### 实例化CompletableFuture

有两种常见的创建 `CompletableFuture` 对象的方法：

1. 通过 `new` 关键字。
2. 基于 `CompletableFuture` 自带的静态工厂方法：`runAsync()`、`supplyAsync()` 。



##### new 关键字

通过 `new` 关键字创建 `CompletableFuture` 对象这种使用方式可以看作是将 `CompletableFuture` 当做 `Future` 来使用。

示例代码：

```java
// 可以将completableFuture看成是一个容器，里面存放异步运算的返回结果
public static final CompletableFuture<String> completableFuture = new CompletableFuture<>();

public static void main(String[] args) throws Exception {
    ExecutorService executorService = Executors.newFixedThreadPool(2);
    
    long start = System.currentTimeMillis();
    
    executorService.execute(() -> {
        try {
            Thread.sleep(600);
            // 在未来某个时刻，线程处理完了最终的结果，将结果放入completableFuture
            completableFuture.complete("i am ready");
        } catch (InterruptedException ignore) {
        }
    });
    
    executorService.execute(() -> {
        while (true) {
            try {
                // 通过completableFuture的isDone方法判断异步运算是否结束
                if (completableFuture.isDone()) {
                    // 从completableFuture中获取异步运算的结果
                    String result = completableFuture.get();
                    System.out.println(result);
                    break;
                }
                Thread.sleep(100);
                System.out.println("wait");
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    });
    
    long end = System.currentTimeMillis();
    System.out.println("耗时：" + (end - start));
    
    executorService.shutdown();
}
```

执行结果：

```tex
耗时：41
wait
wait
wait
wait
wait
wait
i am ready
```



##### 静态工厂方法

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
public static CompletableFuture<Void> runAsync(Runnable runnable);

// 推荐使用下面自定义线程池的方式
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor);
public static CompletableFuture<Void> runAsync(Runnable runnable,Executor executor)
```

静态工厂实例化有两种格式，一种是supply开头的方法，一种是run开头的方法：

- supply开头：方法接收的参数是 `Supplier<U>` ，这也是一个函数式接口，`U` 是返回结果值的类型。当你需要异步操作且关心返回结果的时候，可以使用 `supplyAsync()` 方法。
- run开头：方法接收的参数是 `Runnable` ，这是一个函数式接口，不允许返回值。当你需要异步操作且不关心返回结果的时候可以使用 `runAsync()` 方法。

> 在静态工厂实例化方法中，我们是可以指定Executor参数的，当我们不指定的话，我们所开的并行线程使用的是默认系统及公共线程池 `ForkJoinPool.commonPool()` ，它是被当前  `JVM`（进程）上的所有 `CompletableFuture`、并行 `Stream` 所共享的，`commonPool`  的目标场景是非阻塞的 CPU 密集型任务，其线程数默认为 CPU 数量减1，所以对于我们用 `java` 常做的IO密集型任务，默认线程池是远远不够使用的；
>
> 在双核及以下机器上，默认线程池又会**退化**为给每个任务创建一个线程，相当于没有线程池。
