---
layout: post
title: java-basis-2-hashmap-2
postTitle: Java基础（二）——HashMap剖析（下）
categories: [Java, Software Construction]
description: 软件构造第7次博客
keywords: Java, Software Construction, HashMap, Data Structure
published: true
mathjax: true
typora-root-url: ..
---

## `HashMap`的规约

JavaDocs中`HashMap`的spec是这么写的：

>Hash table based **implementation of the Map interface**. This implementation provides all of the optional map operations, and permits null values and the null key. (The `HashMap` class is roughly equivalent to `Hashtable`, except that it is **unsynchronized** and **permits nulls**.) This class makes no guarantees as to the order of the map; in particular, it does not guarantee that the order will remain constant over time.

这里头可提取出几个关键的信息：

- 基于`Map`接口实现
- 允许`null`键/值
- 非同步
- 不保证有序（如插入的顺序）
- 不保证顺序不随时间变化。

这些问题在上篇都已经基本解释清楚了。下篇中，我们顺着`HashMap`的规约继续往下探索。

## 两个因子

有两个因子直接影响`HashMap`的效率，

- 初始容量（initial capacity）。容量是指哈希表中桶的数量，而初始容量顾名思义就是`HashMap`实例创建时的最初容量。
- 载入因子（Load factor）。含义在上篇已经阐述。

JavaDocs的原文对这两个因子是这样描述的：

>- **Initial capacity**. The capacity is the number of buckets in the hash table, The initial capacity is simply the capacity at the time the hash table is created.
>- **Load factor**. The load factor is a measure of how full the hash table is allowed to get before its capacity is automatically increased.

简单的说，容量（Capacity）就是桶（buckets）的数目，载入因子就是有数据的桶占所有桶的最大比例。如果对迭代性能要求很高的话不要把容量设置过大，也不要把载入因子设置过小。当有数据的桶的数目（即`HashMap`中元素的个数）大于`Capacity * LoadFactor`时就需要调整桶的数目为当前的两倍，也就是上篇所说的扩张。

### 初始容量

`HashMap`的默认初始容量为16。把它设成2的幂次是有意为之。我们再回顾一下index的计算：

```java
index = hash(Key) & (length - 1)
```

因为`length`是$2^k$，所以`length - 1`是$2^k - 1$，也就是$n$个1，index就相当于是`hash(Key)`的最后$k = \log{\textrm{length}}$位。这样，只要保证`hash(Key)`是均匀的，index的分布也就一定是均匀的。此外，这样设计算法使得可以用位运算计算index，避免了取模等效率低下的运算方式，提高了运算效率。

另外，需要注意的是，`hash(Key)`并不是`hashCode(Key)`。`hash`方法的定义如下：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

可以看到这个函数大概的作用就是：高16位不变，低16位和高16位做了一个异或。其中代码注释是这样写的：

>Computes `key.hashCode()` and spreads (XORs) higher bits of hash to lower. Because the table uses power-of-two masking, sets of hashes that vary only in bits above the current mask will always collide. (Among known examples are sets of Float keys holding consecutive whole numbers in small tables.) So we apply a transform that spreads the impact of higher bits downward. There is a tradeoff between **speed, utility, and quality** of bit-spreading. Because many common sets of hashes are already **reasonably distributed** (so don’t benefit from spreading), and because **we use trees to handle large sets of collisions in bins**, we just XOR some shifted bits in the cheapest possible way to reduce systematic lossage, as well as to incorporate impact of the highest bits that would otherwise never be used in index calculations because of table bounds.

也就是说，设计者认为直接用`hashCode(Key)`的值很容易发生碰撞。为什么这么说呢？不妨思考一下，在$n - 1$为15（`0x1111`）时，其实散列真正生效的只是低4位，当然容易碰撞了。

因此，设计者采取了顾全大局的方法（综合考虑了速度、作用、质量），即把高16位和低16位异或了一下（也就是`hash`方法的作用）。设计者还解释到因为现在大多数的`hashCode`的分布已经很不错了，就算是发生了碰撞也用$O(\log{n})$的红黑树去做了（下文会提到）。仅仅异或一下，既减少了系统的开销，也不会造成的因为高位没有参与下标的计算而引起的碰撞。

至于为什么偏偏设成2的4次幂，一般认为没什么特别的理由，用4次幂可能只是因为作者认为16这个初始容量是能符合常用的情景而已。

### 载入因子

上篇中我们提到，载入因子$\alpha$的值默认设为0.75。需要明确的一点是，对于`HashMap`来说，$\alpha$的分母是指总的桶的数量；而分子指的是数据项（即键值对）的数量，不是非空的桶的数量。

为什么设为0.75呢？这也是一个经验值。JavaDocs原文如下：

>As a general rule, the default load factor (.75) offers a good tradeoff between time and space costs.  Higher values decrease the space overhead but increase the lookup cost (reflected in most of the operations of the `HashMap` class, including `get` and `put`).  The expected number of entries in the map and its load factor should be taken into account when setting its initial capacity, so as to minimize the number of rehash operations.  If the initial capacity is greater than the maximum number of entries divided by the load factor, no rehash operations will ever occur.

大意是，负载因子$\alpha$太小了浪费空间并且会发生更多次数的`resize`；$\alpha$太大了会导致哈希冲突增加，性能不好。所以0.75是一个折中的选择，

## 一棵树

之前已经提过，在获取`HashMap`的元素时，基本分两步：

- 首先根据`hashCode()`做`hash`，然后确定桶的index；
- 如果桶的头结点的`Key`不是我们需要的，则通过`Key.equals()`在*链表*中找。

在Java 8之前的实现中是用链表解决冲突的，在产生碰撞的情况下，进行`get`时，两步的时间复杂度最坏情况下是$O(1)+O(n)$，显然是影响速度的。

同一桶内挤进去很多元素的时候，如果仍用链表存储，效率就会非常低下。因此，在桶内元素过多时，Java 8做出了改进：`HashMap`会用红黑树替换链表，将最坏情况下的$O(n)$复杂度降至$O(\log{n})$。那么，一个桶内塞入多少键值对的时候，才会从链表换成红黑树呢？默认情况下，是8个。为什么是8个呢？

这个数值的设置，我们需要平衡以下几个方面：

- **空间性能**。红黑树虽然改善了链表增删改查的性能，但是其结点大小是链表结点的两倍。
- **转换性能**。把链表转换成红黑树的过程需要一定时间成本。
- **数据结构性能**。也就是之前所说的，在冲突较多的情况下，红黑树比链表的性能高很多。

我们考虑数组大小`length`很大的情况。假设哈希函数完全均匀，那么单个桶中的最大结点数满足下面一个泊松分布：`length`个桶，做`0.75 * length`次相同实验，每次实验将一个数据项随机地放入其中一个桶，单个桶内的数据项的数目为$k$的概率就是


$$
P(X=k)=\frac{\lambda^k}{k!}e^{-\lambda}
$$


唯一的问题是$\lambda$是多少？`HashMap`的注释中，直接给出了它的值0.5，但没有给出理由，只说了这个值与载入因子$\alpha$相关：

>Because TreeNodes are about twice the size of regular nodes, we use them only when bins contain enough nodes to warrant use(see TREEIFY_THRESHOLD). And when they become too small (due to removal or resizing) they are converted back to plain bins. In usages with well-distributed user hashCodes, tree bins are rarely used. Ideally, under random hashCodes, the frequency of nodes in bins follows a Poisson distribution(http://en.wikipedia.org/wiki/Poisson_distribution) **with a parameter of about 0.5** on average for the default resizing threshold of 0.75, although with a large variance because of resizing granularity. Ignoring variance, the expected occurrences of list size k are (exp(-0.5) * pow(0.5, k)/factorial(k)). The first values are:
>0: 0.60653066
>1: 0.30326533
>2: 0.07581633
...
>8: 0.00000006
>more: less than 1 in ten million

究竟为什么是0.5呢？以我对泊松分布的理解，这个$\lambda$应该就是0.75才对。我查阅了很多资料，也没有找到一个靠谱的解释。[StackOverflow上的一个帖子](https://stackoverflow.com/questions/20448477/cant-understand-poisson-part-of-hash-tables-from-sun-documentation)称，0.5是`HashMap`作者的又一个假设：

> ... and I think it also assumes that the number of elements in the `HashMap` is 50% of the number of buckets

但从注释的上下文看，这并不是一个假设。而且，就算是假设`HashMap`每个桶存在值的概率是0.5，——那这跟$\lambda$的本意（实验成功次数的期望，或者说事件发生的频率）也并没有关系啊。

恕我愚钝，这里先挖个坑吧。

## 参考资料

- [JavaDocs - HashMap](https://docs.oracle.com/javase/8/docs/api/java/util/HashMap.html)

- [掘金 - HashMap](https://juejin.im/post/5dee6f54f265da33ba5a79c8)

- [WikiPedia - Hash table](https://en.wikipedia.org/wiki/Hash_table)

- [YiKun - HashMap工作原理及实现](https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)

- [ConcurrentHashMap & Hashtable](https://mp.weixin.qq.com/s/AixdbEiXf3KfE724kg2YIw)

- [CSDN - 泊松分布](https://blog.csdn.net/ccnt_2012/article/details/81114920)

- [简书 - HashMap细节](https://www.jianshu.com/p/9ad7a192fd7a)

- [CSDN - LoadFactor默认值和泊松分布没有关系](https://blog.csdn.net/reliveIT/article/details/82960063)

- [StackOverflow - 哈希表与泊松分布](https://stackoverflow.com/questions/20448477/cant-understand-poisson-part-of-hash-tables-from-sun-documentation)

- [简书 - HashMap的loadFactor为什么是0.75？](https://www.jianshu.com/p/64f6de3ffcc1)