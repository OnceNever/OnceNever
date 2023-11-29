---
title: ThreadLocal
date: 2023/10/09 10:42:26
categories:
- [JAVA, 多线程]
tags:
- Thread
- 多线程
---

### ThreadLocal是什么

在处理多线程并发安全的问题中，我们最常使用的方法就是通过锁来控制多个不同线程对临界区的访问。

但无论是什么样的锁，乐观锁或者悲观锁，尽管`JDK` 在升级过程中对它们都有不同程度的优化，但在并发冲突的时候总是会对性能产生一定的影响。

而我们今天的主角 `ThreadLocal` 解决的正是**彻底避免**多线程之间产生的竞争问题。

从字面意思上看，`ThreadLocal` 可以解释为线程的局部变量，也就是说对于同一个 `ThreadLocal` 变量，每个线程访问的都是自己本地变量值，既然只有自己能够访问，那自然就避免了冲突。

所以说，`ThreadLocal` 相较于锁提供了一种与众不同的保证线程安全的方式，它不是在线程发生冲突时想办法解决冲突，而是彻底的避免了冲突的发生。

举一个现实生活当中的例子：

去商场购物时我们总会将购买的商品放到购物车中，并且商场会为每个人准备单独的购物车，如果所有人购买的东西都放到一个购物车，大家就会混淆各自购买的商品，最终结账也会出大问题。所以每个人都有属于自己的购物车，这样才不会各自混淆，`ThreadLocal` 的实现方式也和这个例子类似。



### ThreadLocal使用

`ThreadLocal ` 提供了一个 `withInitial()` 方法统一初始化所有线程的 `ThreadLocal` 的值，这里我们将 `thread1` 、`thread2` 和主线程的值都初始化为0：

```java
public class Test {
    // 创建一个 ThreadLocal 变量
    private final static ThreadLocal<Integer> THREAD_LOCAL = ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) throws InterruptedException {
        // 创建两个线程并启动
        Thread thread1 = new Thread(() -> {
            // 在线程1中设置 ThreadLocal 变量的值
            THREAD_LOCAL.set(1);

            // 在线程1中获取 ThreadLocal 变量的值并打印
            System.out.println("Thread 1 - ThreadLocal Value: " + THREAD_LOCAL.get());
        });

        Thread thread2 = new Thread(() -> {
            // 在线程2中设置 ThreadLocal 变量的值
            THREAD_LOCAL.set(2);

            // 在线程2中获取 ThreadLocal 变量的值并打印
            System.out.println("Thread 2 - ThreadLocal Value: " + THREAD_LOCAL.get());
        });

        // 启动线程1和线程2
        thread1.start();
        thread2.start();

        // 等待线程1和线程2完成
        thread1.join();
        thread2.join();

        // 在主线程中获取 ThreadLocal 变量的值并打印
        System.out.println("Main Thread - ThreadLocal Value: " + THREAD_LOCAL.get());
    }
}
```

执行结果：

```tex
Thread 1 - ThreadLocal Value: 1
Thread 2 - ThreadLocal Value: 2
Main Thread - ThreadLocal Value: 0
```

我们可以看到，`thread1` 和 `thread2` 设置的值互不影响，由于主线程没有重新设置值，所以取得初始化的值0。



### ThreadLocal的实现原理

`ThreadLocal` 变量是如何做到只对当前线程可见的呢？我们先从 `ThreadLocal` 类当中最基本的 `get()` 方法说起：

```java
public T get() {
    // 获得当前线程
    Thread t = Thread.currentThread();
    // ThreadLocalMap里保存着所有ThreadLocal变量
    // 通过getMap(Thread t)方法可以发现 每个线程都有一个自己的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // ThreadLocalMap的key就是当前ThreadLocal的对象实例
        // 多个ThreadLocal变量都是放在这个map中的
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 这里取出来的值就是当前线程对这个ThreadLocal变量设置的值
            T result = (T)e.value;
            return result;
        }
    }
    // 如果map没有初始化，这里执行初始化的操作
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    // 获取当前线程的ThreadLocalMap
    return t.threadLocals;
}
```

可以看到，所谓的 `ThreadLocal` 变量就是保存在每个线程的 `ThreadLocalMap` 中的。这个 `map` 就是 `Thread` 对象中的 `threadLocals` 字段。如下：

```java
// 与此线程有关的ThreadLocal值。由ThreadLocal类维护
ThreadLocal.ThreadLocalMap threadLocals = null;
```

我们可以得出结论，**最终的变量是放在了当前线程的 `ThreadLocalMap` 中，并不是存在 `ThreadLocal` 上，`ThreadLocal` 可以理解为只是 `ThreadLocalMap` 的封装，传递了变量值。**`ThrealLocal` 类中可以通过 `Thread.currentThread()` 获取到当前线程对象后，直接通过`getMap(Thread t)`可以访问到该线程的`ThreadLocalMap`对象。

**每个 `Thread` 中都具备一个 `ThreadLocalMap`，而 `ThreadLocalMap` 可以存储以 `ThreadLocal` 为 key ，Object 对象为 value 的键值对。**

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //......
}
```

比如我们在同一个线程中声明了两个 `ThreadLocal` 对象的话， `Thread` 内部都是使用仅有的那个`ThreadLocalMap` 存放数据的，`ThreadLocalMap`的 key 就是 `ThreadLocal`对象，value 就是 `ThreadLocal` 对象调用`set`方法设置的值。

`ThreadLocal` 结构图如下所示：

![image-20231009141659619](https://image.seeyourface.cn/migrate/image-20231009141659619.png)



### ThreadLocal内存泄漏

`ThreadLocal.ThreadLocalMap` 是一个比较特殊的 `Map` ，它的每个 `Entry` 的 `key` 都是一个弱引用：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;
    // key是一个弱引用
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

`ThreadLocalMap` 中使用的 key 为 `ThreadLocal` 的弱引用，而 `value` 是强引用。所以，如果 `ThreadLocal` 没有被外部强引用的情况下，在垃圾回收的时候，`key` 会被清理掉，而 `value` 不会被清理。

这个 `value` 的引用链条如下：

![ThreadLocalMap中value引用链](https://image.seeyourface.cn/migrate/ThreadLocalMap%E4%B8%ADvalue%E5%BC%95%E7%94%A8%E9%93%BE.jpg)

可以看到，只有当 `Thread` 被回收时，这个 `value` 才有被回收的机会，否则，只要线程不退出，`value` 总是会存在一个强引用。但是，要求每个 `Thread` 都会退出，是一个极其苛刻的要求，对于线程池来说，大部分线程会一直存在在系统的整个生命周期内，那样的话，就会造成 `value` 对象出现泄漏的可能。

如此一来，`ThreadLocalMap` 中就会出现 `key` 为 `null` 的 `Entry`。假如我们不做任何措施的话，`value` 永远无法被 `GC` 回收，这个时候就可能会产生内存泄露。`ThreadLocalMap` 实现中已经考虑了这种情况，在调用 `set()`、`get()`、`remove()` 方法的时候，会清理掉 `key` 为 `null` 的记录。使用完 `ThreadLocal`方法后最好手动调用 `remove()` 方法。

以 `getEntry()` 方法为例：

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        // 如果这个key存在，直接返回结果
        return e;
    else
        // 如果没找到，就会尝试清理，也就是说如果你总是访问存在的key，清理流程永远进不来
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    while (e != null) {
        // 整个e是entry ，也就是一个弱引用
        ThreadLocal<?> k = e.get();
        // 如果找到了，就返回
        if (k == key)
            return e;
        // 如果key为null，说明弱引用已经被回收了
        // 那么就要在这里回收里面的value了
        if (k == null)
            expungeStaleEntry(i);
        else
            // 如果key不是要找的那个，那说明有hash冲突，这里是处理冲突，找下一个entry
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

真正用来回收value的是 `expungeStaleEntry()` 方法，在 `remove()` 和 `set()` 方法中，都会直接或者间接调用到这个方法进行 `value` 的清理：

从这里可以看到，`ThreadLocal` 为了避免内存泄露，也算是花了一番大心思。不仅使用了弱引用维护 `key`，还会在每个操作上检查 `key` 是否被回收，进而再回收 `value` 。

但是从中也可以看到，`ThreadLocal` 并不能100%保证不发生内存泄漏。

比如，很不幸的，你的 `get()` 方法总是访问固定几个一直存在的 `ThreadLocal` ，那么清理动作就不会执行，如果你没有机会调用 `set()` 和 `remove()` ，那么这个内存泄漏依然会发生。

因此，一个良好的习惯依然是：**当你不需要这个 `ThreadLocal` 变量时，主动调用 `remove()`，这样对整个系统是有好处的**。



### ThreadLocalMap中的Hash冲突处理

`ThreadLocalMap` 作为一个 `HashMap` 和 `java.util.HashMap` 的实现是不同的。对于`java.util.HashMap` 使用的是拉链法来处理冲突：

![拉链法](https://image.seeyourface.cn/migrate/%E6%8B%89%E9%93%BE%E6%B3%95.jpg)

但是，对于 `ThreadLocalMap`，它使用的是简单的线性探测法，如果发生了元素冲突，那么就使用下一个槽位存放：

![线性探测法](https://image.seeyourface.cn/migrate/%E7%BA%BF%E6%80%A7%E6%8E%A2%E6%B5%8B%E6%B3%95.jpg)

整个 `set()` 方法过程如下：

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    // 根据key的哈希值对数组长度取模找到一个位置
    int i = key.threadLocalHashCode & (len-1);
    // 如果这个位置没有被占用，说明没有冲突，那就不需要循环，直接使用这个位置
    // 如果发生冲突，那就一直循环往下找，直到找到一个可用的位置
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        // 循环体内，说明已经发生了冲突
        ThreadLocal<?> k = e.get();
        // 如果是对值进行重置，那么直接覆盖就好
        if (k == key) {
            e.value = value;
            return;
        }
        // 如果key为null，说明原来的key被回收了，启动清理
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 一旦找到合适位置，就放入新的Entry
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```



### InheritableThreadLocal

在实际开发过程中，我们可能会遇到这么一种场景。主线程开了一个子线程，但是我们希望在子线程中可以访问主线程中的 `ThreadLocal` 对象，也就是说有些数据需要进行父子线程间的传递。比如像这样：

```java
public class InheritableThreadLocalExample {

    // 创建一个InheritableThreadLocal变量
    private static InheritableThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();

    public static void main(String[] args) {
        // 在主线程中设置值
        inheritableThreadLocal.set("Main Thread Value");

        // 创建一个子线程
        Thread childThread = new Thread(() -> {
            // 子线程可以访问父线程设置的值
            String value = inheritableThreadLocal.get();
            System.out.println("Child Thread Value: " + value);
        });

        // 启动子线程
        childThread.start();

        try {
            // 等待子线程完成
            childThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 主线程仍然可以访问相同的值
        String mainThreadValue = inheritableThreadLocal.get();
        System.out.println("Main Thread Value: " + mainThreadValue);

        // 清除InheritableThreadLocal变量
        inheritableThreadLocal.remove();
    }
}
```

执行上述代码，可以得到结果：

```tex
Child Thread Value: Main Thread Value
Main Thread Value: Main Thread Value
```

可以看到，子线程可以访问到从父进程传递过来的一个数据。虽然 `InheritableThreadLocal` 看起来挺方便的，但是依然要注意以下几点：

1. 变量的传递是发生在线程创建的时候，如果不是新建线程，而是用了线程池里的线程，就不灵了。
2. 变量的赋值就是从主线程的 `map` 复制到子线程，它们的 `value` 是同一个对象，如果这个对象本身不是线程安全的，那么就会有线程安全问题。



### 使用场景

1. 每个线程需要一个独享的对象（通常是工具类，典型需要使用的类有 `SimpleDateFormat` 和`Random` ）。
2. 每个线程内需要保存全局变量（例如在拦截器中获取用户信息），可以让不同方法直接使用，避免参数传递的麻烦。
