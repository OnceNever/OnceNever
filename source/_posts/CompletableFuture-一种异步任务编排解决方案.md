---
title: CompletableFuture-一种异步任务编排解决方案
date: 2023/10/13 15:03:48
categories:
- [JAVA, 多线程]
tags:
- Thread
- 多线程
---

### 概要

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

#### 实例化

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
public static CompletableFuture<Void> runAsync(Runnable runnable);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);

// 推荐使用下面自定义线程池的方式
public static CompletableFuture<Void> runAsync(Runnable runnable,Executor executor);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor);
```

静态工厂实例化有两种格式，一种是supply开头的方法，一种是run开头的方法：

- supply开头：方法接收的参数是 `Supplier<U>` ，这也是一个函数式接口，`U` 是返回结果值的类型。当你需要异步操作且关心返回结果的时候，可以使用 `supplyAsync()` 方法。
- run开头：方法接收的参数是 `Runnable` ，这是一个函数式接口，不允许返回值。当你需要异步操作且不关心返回结果的时候可以使用 `runAsync()` 方法。

> 在静态工厂实例化方法中，我们是可以指定Executor参数的，当我们不指定的话，我们所开的并行线程使用的是默认系统及公共线程池 `ForkJoinPool.commonPool()` ，它是被当前  `JVM`（进程）上的所有 `CompletableFuture`、并行 `Stream` 所共享的，`commonPool`  的目标场景是非阻塞的 CPU 密集型任务，其线程数默认为 CPU 数量减1，所以对于我们用 `java` 常做的IO密集型任务，默认线程池是远远不够使用的；
>
> 在双核及以下机器上，默认线程池又会**退化**为给每个任务创建一个线程，相当于没有线程池。



#### 异步任务回调

##### thenRun()/thenRunAsync()

```java
// 继续沿用上一个任务的线程池
public CompletableFuture<Void> thenRun(Runnable action) {
	return uniRunStage(null, action);
}

// 使用默认线程池 ForkJoinPool.commonPool()
public CompletableFuture<Void> thenRunAsync(Runnable action) {
    return uniRunStage(asyncPool, action);
}

// 使用自定义线程池
public CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor) {
    return uniRunStage(screenExecutor(executor), action);
}
```

接收一个 `Runnable` 参数，即完成某个任务后执行回调方法接着执行第二个任务，前后任务之间没有参数传递，第二个任务也没有返回值。

```java
ExecutorService executorService = Executors.newFixedThreadPool(3);

CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> 
        System.out.printf("%s-执行第一个方法%n", Thread.currentThread().getName()), executorService);

// 沿用上一个任务的线程池
 CompletableFuture<Void> future2 = future1.thenRun(() -> 
         System.out.printf("%s-执行第二个方法%n", Thread.currentThread().getName()));
// 输出
// pool-1-thread-1-执行第一个方法
// pool-1-thread-1-执行第二个方法

// 使用默认的线程池
//CompletableFuture<Void> future2 = future1.thenRunAsync(() ->
//		  System.out.printf("%s-执行第二个方法%n", Thread.currentThread().getName()));
// 输出
// pool-1-thread-1-执行第一个方法
// ForkJoinPool.commonPool-worker-9-执行第二个方法

//ExecutorService executorService2 = Executors.newFixedThreadPool(3);
// 使用自定义线程池
//CompletableFuture<Void> future2 = future1.thenRunAsync(() ->
//        System.out.printf("%s-执行第二个方法%n", Thread.currentThread().getName()), executorService2)
// 输出
// pool-1-thread-1-执行第一个方法
// pool-2-thread-1-执行第二个方法

executorService.shutdown();
```



##### thenAccept()/thenAcceptAsync()

```java
public CompletableFuture<Void> thenAccept(Consumer<? super T> action) {
    return uniAcceptStage(null, action);
}

public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action) {
    return uniAcceptStage(asyncPool, action);
}

public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action,
                                               Executor executor) {
    return uniAcceptStage(screenExecutor(executor), action);
}
```

`Consumer<? super T> action` 表示其可以接受 `T` 类型或 `T` 的超类型的参数。这是为了提高通用性，允许你传递更广泛的类型作为参数。并且回调方法执行完毕后没有返回值。

```java
ExecutorService executorService = Executors.newFixedThreadPool(3);

CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> 1, executorService);

// future1的返回值是 Integer，这里我们可以传入它的超类Number
future1.thenAccept((Number n) -> System.out.println(n.doubleValue() + 10)); // 11.0

executorService.shutdown();
```



##### thenApply()/thenApplyAsync()

```java
public <U> CompletableFuture<U> thenApply(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}

public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(asyncPool, fn);
}

public <U> CompletableFuture<U> thenApplyAsync(
    Function<? super T,? extends U> fn, Executor executor) {
    return uniApplyStage(screenExecutor(executor), fn);
}
```

接收一个入参为 `Function<? super T,? extends U>` 的函数式接口参数，也就是将上一个 `Future` 的返回值当作下一个方法的入参传入，并且回调方法内可以**自定义**返回值。

```java
ExecutorService executorService = Executors.newFixedThreadPool(3);

CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello", executorService);

CompletableFuture<Boolean> future2 = future1.thenApply((str) -> {
	String s = str + " World";
    return s.equals("Hello World");
});

System.out.println(future2.get()); // ture

executorService.shutdown();
```



##### whenComplete()/whenCompleteAsync()

```java
public CompletableFuture<T> whenComplete(
    BiConsumer<? super T, ? super Throwable> action) {
    return uniWhenCompleteStage(null, action);
}

public CompletableFuture<T> whenCompleteAsync(
    BiConsumer<? super T, ? super Throwable> action) {
    return uniWhenCompleteStage(asyncPool, action);
}

public CompletableFuture<T> whenCompleteAsync(
    BiConsumer<? super T, ? super Throwable> action, Executor executor) {
    return uniWhenCompleteStage(screenExecutor(executor), action);
}
```

该方法接收的参数为 `BiConsumer<? super T, ? super Throwable>` ，它接受 `T` 类型或 `T` 的超类型的参数，以及上个方法抛出的异常。

需要注意的是，这个方法并**没有返回**值，`whenComplete` 方法返回的 `CompletableFuture` 的**result是上个任务的结果**。

```java
ExecutorService executorService = Executors.newFixedThreadPool(3);

CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    // 手动抛出异常
    int i = 1 / 0;
    return "hello world";
}, executorService);

CompletableFuture<String> future2 = future1.whenComplete((res, ex) -> {
   if (ex != null) {
       System.out.println(ex.getMessage());
       // java.lang.ArithmeticException: / by zero
   }
});

// 如果我们把手动抛出的异常注释掉，这里的输出将是：future2的结果hello world
System.out.println("future2的结果" + future2.get());

executorService.shutdown();
```



##### handle()/handleAsync()

```java
public <U> CompletableFuture<U> handle(
    BiFunction<? super T, Throwable, ? extends U> fn) {
    return uniHandleStage(null, fn);
}

public <U> CompletableFuture<U> handleAsync(
    BiFunction<? super T, Throwable, ? extends U> fn) {
    return uniHandleStage(asyncPool, fn);
}

public <U> CompletableFuture<U> handleAsync(
    BiFunction<? super T, Throwable, ? extends U> fn, Executor executor) {
    return uniHandleStage(screenExecutor(executor), fn);
}
```

`handle()` 方法和 `whenComplete()` 方法的接收入参没有区别，但不同的地方在于 `handle()` 方法是**有返回值**的，并且是可以自定义返回值的。

`handle(`) 方法返回的 `CompletableFuture` 的 `result` 是**回调方法**执行的结果。

```java
ExecutorService executorService = Executors.newFixedThreadPool(3);

CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    return "future1 result";
}, executorService);

CompletableFuture<String> future2 = future1.handle((res, ex) -> {
    return "future2 result";
});

System.out.println("future2的结果" + future2.get()); // future2的结果future2 result

executorService.shutdown();
```



##### exceptionally()

```java
public CompletableFuture<T> exceptionally(
    Function<Throwable, ? extends T> fn) {
    return uniExceptionallyStage(fn);
}
```

处理任务异常时执行的回调方法，由抛出的异常作为方法入参，有返回值。

```java
ExecutorService executorService = Executors.newFixedThreadPool(3);

CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    int i = 1/0;
    return "future1 result";
}, executorService);

 CompletableFuture<String> exceptionally = future1.exceptionally((ex) -> {
    if (ex != null) {
        System.out.println(ex.getMessage());
        return "程序出现异常";
    }
    return "程序正常运行";
});

System.out.println(exceptionally.get());// 程序出现异常

executorService.shutdown();
```



#### 异步任务组合

##### 总览

![Completable异步任务组合](https://blog.seeyourface.cn/blog/Completable%E5%BC%82%E6%AD%A5%E4%BB%BB%E5%8A%A1%E7%BB%84%E5%90%88.png)

##### AND组合关系

`thenCombine` / `thenAcceptBoth` / `runAfterBoth`都表示：**将两个 `CompletableFuture` 组合起来，只有这两个都正常执行完了，才会执行某个任务**。

它们之间的区别在于：

- `thenCombine`：会将两个任务的执行结果作为方法入参，传递到指定方法中，且**有返回值**。
- `thenAcceptBoth`: 会将两个任务的执行结果作为方法入参，传递到指定方法中，且**无返回值**。
- `runAfterBoth` 不会把执行结果当做方法入参，且没有返回值。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> 5);
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> 7);

// result1 是 future1 的结果，result2 是 future2 的结果
CompletableFuture<Integer> combinedFuture = future1.thenCombine(future2, Integer::sum);

combinedFuture.thenAccept(result -> {
    System.out.println("Combined Result: " + result);
});

// 阻塞等待结果
combinedFuture.join(); // Combined Result: 12
```



##### OR组合关系

`applyToEither` / `acceptEither` / `runAfterEither` 都表示：**将两个 `CompletableFuture` 组合起来，只要其中一个执行完了，就会执行某个任务**。

它们之间的区别在于：

- `applyToEither`：会将已经执行完成的任务，作为方法入参，传递到指定方法中，且**有返回值**

- `acceptEither`：会将已经执行完成的任务，作为方法入参，传递到指定方法中，且**无返回值**

- `runAfterEither`：不会把执行结果当做方法入参，且没有返回值。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 2;
});

CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 4;
});

CompletableFuture<Integer> resultFuture = future1.applyToEither(future2, result -> {
    // result 是首个完成的 CompletableFuture 的结果
    return result * 2;
});

resultFuture.thenAccept(result -> {
    System.out.println("Result of the first completed future: " + result);
});

// 阻塞等待结果
resultFuture.join(); // 8
```



##### allOf()

`allOf` 方法用于等待多个 `CompletableFuture` 都完成后执行操作。它不返回一个合并的结果，而只是在所有的 `CompletableFuture` 都完成后触发一个操作。这个方法通常用于等待多个任务都完成后才继续执行其他操作。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("future1 is done");
    return 5;
});

CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("future2 is done");
    return 7;
});

CompletableFuture<Integer> future3 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("future3 is done");
    return 3;
});

CompletableFuture<Void> allOfFuture = CompletableFuture.allOf(future1, future2, future3);

allOfFuture.thenRun(() -> {
    System.out.println("All futures have completed.");
});

// 阻塞等待所有的 CompletableFuture 完成
allOfFuture.join();

// 输出
// future3 is done
// future1 is done
// future2 is done
// All futures have completed.
```



##### anyOf()

`anyOf` 方法用于等待多个 `CompletableFuture` 中的任何一个完成后执行操作。它不等待所有 `CompletableFuture` 都完成，只需等待任何一个完成即可触发操作。这对于在多个任务中获取最快完成的结果非常有用。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 5;
});

CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(3);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 7;
});

CompletableFuture<Integer> future3 = CompletableFuture.supplyAsync(() -> {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 3;
});

CompletableFuture<Object> anyOfFuture = CompletableFuture.anyOf(future1, future2, future3);
anyOfFuture.thenAccept(result -> {
    System.out.println("Result of the first completed future: " + result);
});

// 阻塞等待结果
anyOfFuture.join(); // Result of the first completed future: 3
```



##### thenCombine()

`thenCombine` 方法是用于组合两个独立的 `CompletableFuture` 的结果，然后在两者都完成时执行一个操作。

这个操作接受两个参数，分别是两个 `CompletableFuture` 的结果，并返回一个新的结果。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> 5);
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> 7);

// result1 是 future1 的结果，result2 是 future2 的结果
CompletableFuture<Integer> combinedFuture = future1.thenCombine(future2, Integer::sum);

combinedFuture.thenAccept(result -> {
	System.out.println("Combined Result: " + result);
});

// 阻塞等待结果
combinedFuture.join(); //Combined Result: 12
```



### 注意事项

#### Future需要获取返回值，才能获取异常信息。

```java
ExecutorService executorService = new ThreadPoolExecutor(5, 10, 5L,
    TimeUnit.SECONDS, new ArrayBlockingQueue<>(10));
CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
      int a = 0;
      int b = 666;
      int c = b / a;
      return true;
   },executorService).thenAccept(System.out::println);
   
 //如果不加 get()方法这一行，看不到异常信息
 //future.get();
```



#### 尽量避免使用 get()

`CompletableFuture`的`get()`方法是阻塞的，尽量避免使用。如果必须要使用的话，需要添加超时时间，否则可能会导致主线程一直等待，无法执行其他任务。

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(10_000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "Hello, world!";
});

// 获取异步任务的返回值，设置超时时间为 5 秒
try {
    String result = future.get(5, TimeUnit.SECONDS);
    System.out.println(result);
} catch (InterruptedException | ExecutionException | TimeoutException e) {
    // 处理异常
    e.printStackTrace();
   }
}
```

上面这段代码在调用 `get()` 时抛出了 `TimeoutException` 异常。这样我们就可以在异常处理中进行相应的操作，比如取消任务、重试任务、记录日志等。



#### 使用自定义线程池

`CompletableFuture` 默认使用`ForkJoinPool.commonPool()` 作为执行器，这个线程池是全局共享的，可能会被其他任务占用，导致性能下降或者饥饿。因此，建议使用自定义的线程池来执行 `CompletableFuture` 的异步任务，可以提高并发度和灵活性。



#### 正确进行异常处理

使用 `CompletableFuture`的时候一定要以正确的方式进行异常处理，避免异常丢失或者出现不可控问题。

下面是一些建议：

- 使用 `whenComplete` 方法可以在任务完成时触发回调函数，并正确地处理异常，而不是让异常被吞噬或丢失。
- 使用 `exceptionally` 方法可以处理异常并重新抛出，以便异常能够传播到后续阶段，而不是让异常被忽略或终止。
- 使用 `handle` 方法可以处理正常的返回结果和异常，并返回一个新的结果，而不是让异常影响正常的业务逻辑。
- 使用 `CompletableFuture.allOf` 方法可以组合多个 `CompletableFuture`，并统一处理所有任务的异常，而不是让异常处理过于冗长或重复。



#### 合理组合多个异步任务

正确使用 `thenCompose()` 、 `thenCombine()` 、`acceptEither()`、`allOf()`、`anyOf() `等方法来组合多个异步任务，以满足实际业务的需求，提高程序执行效率。

实际使用中，我们还可以利用或者参考现成的异步任务编排框架，比如京东的 [asyncTool](https://gitee.com/jd-platform-opensource/asyncTool) 。
