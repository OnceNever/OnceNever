---
title: 为什么ConcurrentHashMap的key/value不能为空，HashMap可以？
date: 2023/9/23 22:14:31
categories:
- [JAVA, 集合]
tags:
- ConcurrentHashMap
- HashMap
---

### 为什么ConcurrentHashMap的key/value不能为空，HashMap可以？

以JDK8为例，在ConcurrentHashMap源码的`putVal`方法中当传入的参数key或者value为null时，会抛出`NullPointerException`

<img src="https://blog.seeyourface.cn/blog/image-20230923222339494.png" alt="image-20230923222339494" style="zoom: 67%;" />

`ConcurrentHashMap` 的 key 和 value 不能为 null 主要是为了避免二义性。

null 是一个特殊的值，表示没有对象或没有引用。如果你用 null 作为键，那么你就无法区分这个键是否存在于 ConcurrentHashMap 中，还是根本没有这个键。同样，如果你用 null 作为值，那么你就无法区分这个值是否是真正存储在 ConcurrentHashMap 中的，还是因为找不到对应的键而返回的。



拿 get 方法取值来说，返回的结果为 null 存在两种情况：

- 值没有在集合中 ；
- 值本身就是 null。

这也就是二义性的由来。



多线程环境下，**存在**一个线程操作该 `ConcurrentHashMap` 时，其他的线程将该 `ConcurrentHashMap` 修改的情况，所以**无法通过** `containsKey(key)` 来判断否存在这个键值对，也就没**办法解决二义性**问题了。

与此形成对比的是，`HashMap` 可以存储 null 的 key 和 value，但 null 作为键只能有一个，null 作为值可以有多个。如果传入 null 作为参数，就会返回 hash 值为 0 的位置的值。单线程环境下，**不存在**一个线程操作该 `HashMap` 时，其他的线程将该 `HashMap` 修改的情况，所以可以通过 `contains(key)`来做判断是否存在这个键值对，从而做相应的处理，也就不存在二义性问题。

也就是说，**多线程下无法正确判定键值对是否存在**（存在其他线程修改的情况），单线程是可以的（不存在其他线程修改的情况）。

如果确实需要在 `ConcurrentHashMap` 中使用 null 的话，可以使用一个特殊的静态空对象来代替 null。

```java
public static final Object NULL = new Object();
```

---

关于`ConcurrentHashMap` 作者本人 (Doug Lea)对于这个问题做出的回答：

> The main reason that nulls aren't allowed in ConcurrentMaps (ConcurrentHashMaps, ConcurrentSkipListMaps) is that ambiguities that may be just barely tolerable in non-concurrent maps can't be accommodated. The main one is that if map.get(key) returns null, you can't detect whether the key explicitly maps to null vs the key isn't mapped. In a non-concurrent map, you can check this via map.contains(key), but in a concurrent one, the map might have changed between calls.

翻译：

ConcurrentMaps（ConcurrentHashMaps、ConcurrentSkipListMaps）中不允许使用 null 的主要原因是，无法容纳在非并发映射中勉强可以容忍的歧义。主要的一点是，如果map.get(key)返回null，你无法检测该键是否显式映射到null与该键是否未映射。在非并发映射中，您可以通过 map.contains(key) 检查这一点，但在并发映射中，映射可能在**调用之间**发生了变化。

#### 复合操作

那么**问题来了**，你有可能会问：八股我都背烂了`ConcurrentHashMap`不是线程安全的吗，为什么会有其他线程造成影响？

`ConcurrentHashMap` 是线程安全的，意味着它可以保证**多个线程同时对它进行读写操作**时，不会出现数据不一致的情况，也不会导致 JDK1.7 及之前版本的 HashMap 多线程操作导致死循环问题。但是，这并不意味着它可以保证所有的复合操作都是原子性的，一定不要搞混了！

复合操作是指由多个**基本操作**(如put、get、remove、containsKey等)组成的操作，例如先判断某个键是否存在**containsKey(key)**，然后根据结果进行插入或更新**put(key, value)**。这种操作在执行过程中可能会被其他线程**打断**，导致结果不符合预期。

例如，有两个线程 A 和 B 同时对 `ConcurrentHashMap` 进行复合操作，如下：

<img src="https://blog.seeyourface.cn/blog/image-20230923224919178.png" alt="image-20230923224919178" style="zoom: 67%;" />

如果线程 A 和 B 的执行顺序是这样：

1. 线程 A 判断 map 中不存在 key
2. 线程 B 判断 map 中不存在 key
3. 线程 B 将 (key, anotherValue) 插入 map
4. 线程 A 将 (key, value) 插入 map

那么最终的结果是 (key, value)，而不是预期的 (key, anotherValue)。这就是**复合操作的非原子性**导致的问题。

#### 那如何保证 ConcurrentHashMap 复合操作的原子性呢？

`ConcurrentHashMap` 提供了一些原子性的复合操作，如 putIfAbsent、compute、computeIfAbsent 、computeIfPresent、merge等。

这些方法都可以接受一个函数作为参数，根据给定的 key 和 value 来计算一个新的 value，并且将其更新到 map 中。

<img src="https://blog.seeyourface.cn/blog/image-20230923225542922.png" alt="image-20230923225542922" style="zoom:80%;" />