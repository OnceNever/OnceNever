---
title: JDK源码系列-HashMap
date: 2023/9/27 14:30:26
categories:
- [源码, JDK]
tags:
- 哈希表
- HashMap
---

### 介绍

`HashMap`是Java程序员使用频率最高的用于映射(key-value键值对)处理的数据类型。

JDK1.8 之前 `HashMap` 是由 数组+链表 组成的，数组是 `HashMap` 的主体，链表则是主要为了解决**哈希冲突**而存在的（“拉链法”解决冲突）。 

JDK1.8 以后的 `HashMap` 在解决哈希冲突时有了较大的变化，当链表长度**大于等于阈值**（默认为 8）（将链表转换成红黑树前会判断，如果当前**数组的长度小于 64**，那么会选择**先进行数组扩容**，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间。

`HashMap`的优点是**访问速度快**，插入和删除操作也方便。一般来说，如果`HashMap`中的元素是均匀分布在数组中的，那么查询时间复杂度接近于O(1)；相反，那么查询的时间复杂度可能会增加，因为可能需要遍历数组中的链表或红黑树来找到对应的值。链表的查询时间复杂度是O(n)，红黑树的查询时间复杂度是O(logn)，其中n是链表或红黑树中的元素个数。因此，`HashMap`的查询时间复杂度最好是O(1)，最坏是O(n)。

`HashMap`的缺点是不保证元素的顺序，不支持线程同步，也不能存储重复的键（key）。

`HashMap`可以存储 null 的 key 和 value，但 null 作为键只能有一个，null 作为值可以有多个。

`HashMap` 默认的初始化大小为 16。之后每次扩充，容量变为原来的 2 倍。并且，`HashMap`总是使用 2 的幂作为哈希表的大小。



### 示意图

下图是`HashMap`的类结构关系图：

它继承了`AbstractMap`，`AbstractMap`实现了顶层接口Map中的大部分方法，只有一个抽象方法`entrySet()`需要自己实现。

实现了以下接口：

- Map：哈希表的顶级接口，定义了哈希表的基础操作方法，交给子类实现。
- Cloneable：表明它具有拷贝能力，可以进行深拷贝或浅拷贝操作。
- Serializable：表明它可以进行序列化和反序列化操作，也就是即可以将**对象序列化为字节流**进行持久化存储或网络传输，也可以从**字节流反序列化为对象**，非常方便。

![image-20230927152729707](https://blog.seeyourface.cn/blog/image-20230927152729707.png)



### 底层数据结构

从底层数据存储结构实现来讲，`HashMap`是数组 + 链表 + 红黑树（JDK1.8增加了红黑树部分）实现的，如下如所示。



![hashMap](https://blog.seeyourface.cn/blog/hashMap.png)



大家有没有想过：`HashMap`数据底层具体存储的是什么？这样的存储方式有什么优点呢？

问题1：通过源码我们可以发现，`HashMap`的数据最终都保存在一个叫`Node<K,V>`的结构中。

什么！还有高手？？？



![image-20230927170909332](https://blog.seeyourface.cn/blog/image-20230927170909332.png)



`Node`是`HashMap`的一个内部类，实现了`Map.Entry`接口，本质是就是一个映射(键值对)。也就是上图中的长方形代表的结构。



问题2：哈希表为了解决冲突，可以采用的解决方法有开放地址法和链地址法，且`Java`中`HashMap`采用的就是链地址法。简单来说就是数组加链表的结合。在每个数组元素上都一个链表结构，当数据被`Hash`后，得到数组下标，把数据放在对应下标元素的链表上。



### HashMap是如何保证高性能的？

前面我们说过，理想情况下`HashMap`的查询时间复杂度是O(1)，但是随着加入的对象越来越多，数组中的链表将越来越长，将严重影响HashMap的性能。

那么如何降低哈希冲突的概率呢，可以通过控制变量法进行分析：我们知道一个对象存放的位置取决于**散列函数**和**数组容量**。

- 我们将哈希桶的容量固定，Hash算法越好，对象在哈希桶中的位置分布就越均匀，也就是哈希冲突的概率越低。
- 对于同一个Hash算法，哈希桶越大，对象在哈希桶中冲突的概率也越低。

如果哈希桶数组很大，即使较差的Hash算法也会比较分散，如果哈希桶数组数组很小，即使好的Hash算法也会出现较多碰撞，所以就需要在空间成本和时间成本之间权衡，其实就是在根据实际情况确定哈希桶数组的大小，并在此基础上设计好的Hash算法减少Hash碰撞。

而`HashMap`正是通过好的Hash算法和哈希桶的扩容机制来平衡空间与时间之间的关系。



### 什么时候触发扩容机制

在介绍Hash算法和扩容流程之前，我想先提一下 `HashMap` 在什么时候会触发扩容机制。

通过查看`HashMap`源码可知，扩容机制是通过 `resize()` 方法实现的，而我们在往 `HashMap` 中放入对象时，满足条件 `(++size > threshold)` 时会调用 `resize()` 方法进行扩容，`threshold` 在源码中给出的解释是：在给定 `Load factor` 和 `capacity` (数组容量)下所允许的最大元素数目，即 `threshold = capacity * load factor ` ，默认的负载因子(`load factor`)是0.75。

如果使用无参构造器创建 `HashMap` 默认容量为 `1 >> 4` 即16，所以当添加第13个对象时就会触发`HashMap`的扩容机制。



![image-20230928134857618](https://blog.seeyourface.cn/blog/image-20230928134857618.png)



也就是说，在数组定义好长度之后，负载因子越大，所能容纳的键值对个数越多，超过这个数目就重新resize(扩容)。

默认的负载因子0.75是对空间和时间效率的一个平衡选择，建议大家不要修改，除非在时间和空间比较特殊的情况下，如果内存空间很多而又对时间效率要求很高，可以适当降低负载因子 `loadFactor` 的值；相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子 `loadFactor` 的值，这个值**可以大于**1。



### 为什么HashMap的容量始终为2的N次幂

`HashMap`扩容后的容量是之前容量的两倍。并且在 `HashMap` 中，哈希桶数组table的容量大小始终为2的n次方(一定是合数)。这是一种非常规的设计，常规的设计是把桶的大小设计为素数。相对来说素数导致冲突的概率要小于合数，例如 `Hashtable` 初始化桶大小为11，就是桶大小设计为素数的应用（`Hashtable`扩容后不能保证还是素数）。

`HashMap` 采用这种非常规设计，主要是为了在**取模和扩容**时做优化。

如下图在容量N为 `2 ^ 3 = 8` 的 `HashMap` 中，我们举个例子来证明素数的冲突概率要小于合数：



![image-20230928145903047](https://blog.seeyourface.cn/blog/image-20230928145903047.png)



首先，二进制中用位运算来执行取模操作，只适用于模数为**2的倍数**的情况，这里解释了`HashMap`用2的N次幂做容量的第一个原因。

K = 28 对 N 取模的运算 28 % N 可以转化为位运算的 `1 1100 & (N - 1)` ，上面的例子结果就是4；再对另一个元素 K = 20 做同样运算，我们能得到该元素最终存储的索引也是4。

通过上图我们发现，即使 28 和 20 转化为二进制后的第四位（从右往左数）不相同，但仍然哈希到了同一个位置，也就是说这时候元素K第四位就根本不参与哈希运算，这就无法完整地反映元素 K 的特性，增大了导致冲突的几率。

取其他合数时，都会不同程度的导致c的某些位”失效”，从而在一些常见应用中导致冲突。

但是取质数，基本可以保证K的每一位都参与哈希运算，从而在常见应用中减小冲突几率（并不能完全避免）。

所以为了减少冲突，`HashMap` 定位哈希桶索引位置时，也加入了**高位参与运算**的过程。



### 链表和红黑树的相互转化

即使负载因子和 `Hash` 算法设计的再合理，也无法避免会出现拉链过长的情况，一旦出现拉链过长，则会严重影响 `HashMap` 的性能。

于是，在 `JDK1.8` 版本中，对数据结构做了进一步的优化，引入了红黑树。而当链表长度太长超过 `TREEIFY_THRESHOLD`（默认为8）时，会进一步判断数组容量，**如果数组容量小于 `MIN_TREEIFY_CAPACITY` (默认为64)，会优先对数组进行扩容**，然后将数据重新散列到新的哈希桶中；如果数组容量大于等于64，就会将链表转换为红黑树，利用红黑树快速增删改查的特点提高 `HashMap` 的性能。

在扩容过程中，如果原来数组中红黑树的节点经重新散列后**小于等于** `UNTREEIFY_THRESHOLD` (默认为6)时，红黑树会重新退化为链表。



![img-20230928153456](https://blog.seeyourface.cn/blog/img-20230928153456.png)



### 源码分析

#### 构造方法

```java
// 默认构造函数。
public HashMap() {
   this.loadFactor = DEFAULT_LOAD_FACTOR; // all   other fields defaulted
}

// 包含另一个“Map”的构造函数
public HashMap(Map<? extends K, ? extends V> m) {
   this.loadFactor = DEFAULT_LOAD_FACTOR;
   putMapEntries(m, false);//下面会分析到这个方法
}

// 指定“容量大小”的构造函数
public HashMap(int initialCapacity) {
   this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

// 指定“容量大小”和“负载因子”的构造函数
public HashMap(int initialCapacity, float loadFactor) {
   if (initialCapacity < 0)
       throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
   if (initialCapacity > MAXIMUM_CAPACITY)
       initialCapacity = MAXIMUM_CAPACITY;
   if (loadFactor <= 0 || Float.isNaN(loadFactor))
       throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
   this.loadFactor = loadFactor;
   // 初始容量暂时存放到 threshold ，在resize中再赋值给 newCap 进行table初始化
   this.threshold = tableSizeFor(initialCapacity);
}
```



> 值得注意的是：上述四个构造方法中，都初始化了负载因子 `loadFactor`，由于`HashMap`中没有 `capacity` 这样的字段，即使指定了初始化容量 `initialCapacity` ，也只是通过 `tableSizeFor` 将其扩容到与 `initialCapacity` **最接近的2的幂次方大小**，然后暂时赋值给 `threshold` ，后续通过 `resize` 方法将 `threshold` 赋值给 `newCap` 进行 `table` 的初始化。



#### tableSizeFor()

```java
// Returns a power of two size for the given target capacity.
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```



对任意十进制数转换为2的整数幂，结果是这个数本身的**最高有效位的前一位变成1，最高有效位以及其后的位都变为0**。

核心思想是，先**将最高有效位以及其后的位都变为1**，最后再+1，就进位到前一位变成1，其后所有的满2变0。所以关键是**如何将最高有效位后面都变为1**。

- 右移一位，再或运算，最高有效位就有两位变为1
- 右移两位，再或运算，最高有效位就有四位变为1
- 右移16位再或运算，保证32位的int类型整数最高有效位之后的位都能变为1。
- 最后加1就能达到想要的效果。

开始移位前先将容量先减1，是为了避免给定容量已经是8, 16这样已经是2的幂数时，不减一直接移位会导致得到的结果比预期大。比如预期16得到应该是16，直接移位的话会得到32。



#### putMapEntries()

```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        // 判断table是否已经初始化
        if (table == null) { // pre-size
            /*
             * 未初始化，s为m的实际元素个数，ft = s/loadFactor => s=ft*loadFactor, 跟我们前面提到的
             * 阈值=容量*负载因子 是不是很像，ft指的是要添加s个元素所需的最小的容量。
             */
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                    (int)ft : MAXIMUM_CAPACITY);
            /*
             * 根据构造函数可知，table未初始化，threshold实际上是存放的初始化容量，如果添加s个元素所
             * 需的最小容量大于初始化容量，则将最小容量扩容为最接近的2的幂次方大小作为初始化。
             * 注意这里不是初始化阈值
             */
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        // 已初始化，并且m元素个数大于阈值，进行扩容处理
        else if (s > threshold)
            resize();
        // 将m中的所有元素添加至HashMap中，如果table未初始化，putVal中会调用resize初始化或扩容
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```



#### Hash算法

无论是增加、删除还是查找键值对，定位到哈希桶数组的位置都是很关键的第一步。前面说过 `HashMap` 的数据结构是数组和链表的结合，所以我们当然希望这个 `HashMap` 里面的元素位置尽量分布均匀些，尽量使得每个位置上的元素数量只有一个，那么当我们用hash算法求得这个位置的时候，马上就可以知道对应位置的元素就是我们要的，不用遍历链表，大大优化了查询的效率。

hash方法的离散性能直接决定了 `HashMap` 定位数组索引位置效率，我们看看源码是如何实现的：



```java
// JDK1.8
// 首先取得hashCode的值 h
// h 与 h 无符号向右位移16位做异或运算，目的是让高位参与运算，而异或运算保证0 1出现的概率相等
static final int hash(Object key) {
	int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// JDK 1.7
// 相比于 JDK1.8 的 hash 方法 ，JDK 1.7 的 hash 方法的性能会稍差一点点，因为毕竟扰动了 4 次。
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).

    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```



#### 确定元素在哈希桶中的位置

对于任意给定的对象，只要它的 `hashCode()` 返回值相同，那么程序调用方法一所计算得到的Hash码值总是相同的。

我们首先想到的就是把hash值对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。

但是，模运算的消耗还是比较大的，在 `HashMap` 中是这样做的：`HashMap` 底层数组的长度总是2的n次方，这是 `HashMap` 在速度上的优化。**当length总是2的n次方时**，`h & (length-1)` 运算**等价于**对 `length` 取模，也就是 `h % length`，但是**&比%具有更高的效率**。



#### put()方法

`HashMap` 只提供了 put 用于添加元素，putVal 方法只是给 put 方法调用的一个方法，并没有提供给用户使用。

**对 putVal 方法添加元素的分析如下：**

1. 如果定位到的数组位置没有元素 就直接插入。
2. 如果定位到的数组位置有元素就和要插入的 key 比较，如果 key 相同就直接覆盖，如果 key 不相同，就判断 p 是否是一个树节点，如果是就调用`e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value)`将元素添加进入。如果不是就遍历链表插入(插入的是链表尾部)。



![image-20230928175621083](https://blog.seeyourface.cn/blog/image-20230928175621083.png)



画的有点乱，大家将就看



```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table未初始化或者长度为0，进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 桶中已经存在元素（处理hash冲突）
    else {
        Node<K,V> e; K k;
        //快速判断第一个节点table[i]的key是否与插入的key一样，若相同就直接使用插入的值p替换掉旧的值e。
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
        // 判断插入的是否是红黑树节点
        else if (p instanceof TreeNode)
            // 放入树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 不是红黑树节点则说明为链表结点
        else {
            // 在链表最末插入结点
            for (int binCount = 0; ; ++binCount) {
                // 到达链表的尾部
                if ((e = p.next) == null) {
                    // 在尾部插入新结点
                    p.next = newNode(hash, key, value, null);
                    // 结点数量达到阈值(默认为 8 )，执行 treeifyBin 方法
                    // 这个方法会根据 HashMap 数组来决定是否转换为红黑树。
                    // 只有当数组长度大于或者等于 64 的情况下，才会执行转换红黑树操作，以减少搜索时间。否则，就是只是对数组扩容。
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    // 跳出循环
                    break;
                }
                // 判断链表中结点的key值与插入的元素的key值是否相等
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 相等，跳出循环
                    break;
                // 用于遍历桶中的链表，与前面的e = p.next组合，可以遍历链表
                p = e;
            }
        }
        // 表示在桶中找到key值、hash值与插入元素相等的结点
        if (e != null) {
            // 记录e的value
            V oldValue = e.value;
            // onlyIfAbsent为false或者旧值为null
            if (!onlyIfAbsent || oldValue == null)
                //用新值替换旧值
                e.value = value;
            // 访问后回调
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // 结构性修改
    ++modCount;
    // 实际大小大于阈值则扩容
    if (++size > threshold)
        resize();
    // 插入后回调
    afterNodeInsertion(evict);
    return null;
}
```



#### get()方法

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 数组元素相等
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 桶中不止一个节点
        if ((e = first.next) != null) {
            // 在树中get
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 在链表中get
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```



#### resize()方法

进行扩容，会伴随着一次重新 hash 分配，并且会遍历 hash 表中所有的元素，是非常耗时的。在编写程序中，要尽量避免 resize。resize方法实际上是将 table 初始化和 table 扩容 进行了整合，底层的行为都是给 table 赋值一个新的数组。



```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 超过最大值就不再扩充了，就只好随你碰撞去吧
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 没超过最大值，就扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        // 创建对象时初始化容量大小放在threshold中，此时只需要将其作为新的数组容量
        newCap = oldThr;
    else {
        // signifies using defaults 无参构造函数创建的对象在这里计算容量和阈值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        // 创建时指定了初始化容量或者负载因子，在这里进行阈值初始化，
    	// 或者扩容前的旧容量小于16，在这里计算新的resize上限
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 把旧哈希表中的每个bucket都移动到新的buckets中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 只有一个节点，直接计算元素新的位置即可
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 将红黑树拆分成2棵子树，拆分后的子树节点数小于等于6，则将树转化成链表
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else {
                    // 定义低位链表的头节点和尾节点
                    Node<K,V> loHead = null, loTail = null;
                    // 定义高位链表的头节点和尾节点
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 判断原hash值有效位的高一位是0，连接到低位链表，并保持元素的相对顺序不变，节省对新数组长度重新取模的时间
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 判断原hash值有效位的高一位是0，连接到高位链表，并保持元素的相对顺序不变，节省对新数组长度重新取模的时间  
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 低位链表的头节点连接到原索引位置的bucket里
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 高位链表的头节点连接到原索引位置 + oldCap的bucket里
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

