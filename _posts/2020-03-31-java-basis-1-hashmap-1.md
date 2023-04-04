---
layout: post
title: java-basis-1-hashmap-1
postTitle: Java基础（一）——HashMap剖析（上）
categories: [Java, Software Construction]
description: 软件构造第6次博客
keywords: Java, Software Construction, HashMap, Data Structure
published: true
mathjax: true
typora-root-url: ..
---

## `HashMap`与`Node`

JDK中为我们提供了`HashMap`这一数据结构，声明如下，

```java
public class HashMap<K,V> 
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

它本质上是一个哈希表，且可以在常数时间内完成`get`和`put`操作。`HashMap`采用的是数组+链表的实现，如下图所示：

![315px-Hash_table_3_1_1_0_1_0_0_SP.svg](https://i.loli.net/2021/06/15/hwgd3cQqzfL7WEB.png)

数组中的每个桶都存储了一个`<Key, Value>`键值对结点。这种结点Java 8以上被称作`Node`。每个`Node`结点都会保存自己的`hash`、`key`和`value`，源码如下：

```java
/**
     * Basic hash bin node, used for most entries.
*/
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
		...
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        ...
    }

    public final boolean equals(Object o) {
		...
    }
}
```

初始情况下，数组中的所有位置都为空。用`put`方法插入时，会用`Key`的`hashCode`方法计算其哈希值，作为哈希表中的index。如果`index`对应的「桶」（即数组位置）已经被占用了，且新插入的键值对的键与那个位置上的已有键都不同，就说明发生了冲突。遇到这种情况，就在已有节点上往下挂一个新节点，存储新的键值对。这样，就形成了链表结构。可以看到，`Node`类中还有一成员`next`，它就是指向同一桶内下一个键值对的引用。`get`方法查找时，找到`Key`对应的桶后，就从头遍历链表，找到相应的键值对结点，获得其`Value`并返回。

## 键值对的加入

### 扩容`resize`

扩容是动态数据结构常用的控制大小的方式。最著名的例子莫过于C++中的`vector`。一般来说，数据结构类会设定两个常数值`$\alpha$`和`$\gamma$``$(0<\alpha,\gamma<1)$`，功能分别是：

- `$\alpha$`是**装载因子**，当存储的数据项的数量超过当前容量的`$\alpha$`比例时，就将新建一个数组，但容量扩大一倍。然后，把原数组的内容重新哈希到经过扩容的新数组中去。注意，这里不能直接拷贝过去，因为index的计算是跟`HashMap`的大小相关的：

  ```java
  index = hash(Key) & (Length - 1)
  ```

  这个公式的设计是非常巧妙的。我们发现，键值对的新位置要么是在原位置，要么是在原位置的基础上再移动2次幂个的位置。这样，在扩充`HashMap`的时候，就不用真的把每个键值对的index都重新算一遍了，大幅提升了时间效率。

- `$\gamma$`一般设为`$\alpha$`的一半，称为数据量的下界。它表示数据项的数量低于当前容量的`$\gamma$`比例时，就把容量缩小为原先的一半。

对于`HashMap`来说，初始的`$\alpha$`值是0.75。

### 头插还是尾插

一个需要注意的细节是，Java 8之前，插入新结点时，都是优先在头部插入，因为作者认为新插入的键值对更有可能被先访问到，因此头部插入的时间效率可能更高。但是，Java 8及之后，`HashMap`的实现就改成了从尾部插入。为何要做这样的改变呢？

原因其实相当微妙。下面是Java 7的`HashMap`在扩容时调用的`transfer`方法，用于将原数组中的内容转移到扩容后的新数组中去。

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            // 头插法
            // 把自己的next置为新桶的头元素
            e.next = newTable[i];
            // 把新桶的头元素置为自己
            newTable[i] = e;
            // 继续遍历原桶中的下一个元素
            e = next;
        }
    }
}
```

不难看出，在转移元素的过程中，使用的是头插法。这意味着后加入的元素反而在上面，也就是链表的顺序会发生翻转。这会造成什么问题呢？不妨先看下面这个例子。

先做一些简化问题的假设：

- 哈希算法为简单的用`Key`对数组大小取余。
- 最开始哈希表大小为2，`Key` = [3, 7, 5]，因此都在1号桶中。
- 然后我们进行`resize`，使数组大小翻倍为4。

未`resize`前的`HashMap`如下：

<img src="https://i.loli.net/2021/06/15/uQcqZA9rvKk2WV3.png" alt="706569-20190228162257774-1437530387" style="zoom:67%;" />

经过`resize`后的，键值对的位置变成`Key`对4取余。这时的`HashMap`如下：

<img src="https://i.loli.net/2021/06/15/nsciBq2fwW6YKIO.png" alt="image-20200410183126804" style="zoom:67%;" />

下面考虑一个多线程环境（关于多线程的更系统性的内容，将在后续博客中更新）。假设线程A插入`Key`为3的元素，线程B也插入`Key`为3的元素。然后，线程A在执行到这一行的时候被挂起：

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            ...
            e.next = newTable[i];
             // 线程A在第一次执行到此处时就被挂起
            newTable[i] = e; 
            e = next;
        }
    }
}
```

此时线程A内存中的情况如下图所示：

<img src="D:\Work\blogs\ArmeriaWang.github.io\images\asserts\706569-20190228225719031-1717698945.png" alt="706569-20190228225719031-1717698945" style="zoom:67%;" />

线程A挂起后，线程B正常执行，并完成`resize`操作，结果如下：

<img src="https://i.loli.net/2021/06/15/h4mVY9GX1z7qwkb.png" alt="706569-20190228225719031-1717698945" style="zoom:67%;" />

由于线程B已经执行完毕，根据Java内存模型，现在`newTable`和`table`中的键值对都是主存中最新值：`7.next=3`，`3.next=null`。

然后，被挂起的线程A被重新拎起来执行。别忘了此时线程A的内存的值为：`i` = 3，`e` = 3，`next` = 7，`newTable[i]` = `newTable[3]` = `null`。接下来的两行代码，将会发生的事情是：

```java
newTable[i] = e  // => newTable[3] = 3
e = next         // => e = 7
```

这意味着，原本已经被线程B整理好的链表又被A打乱了。现在主存中情形如下图所示：

<img src="https://i.loli.net/2021/06/15/ItM8yBSefRn2Vcz.png" alt="706569-20190228173450442-1058177260" style="zoom:67%;" />

接下来，线程A继续循环。整个`transfer`方法执行完毕后，我们会得到这样的一个哈希表：

<img src="D:\Work\blogs\ArmeriaWang.github.io\images\asserts\706569-20190228174116585-716787662.png" alt="706569-20190228174116585-716787662" style="zoom:67%;" />

不幸地发生了死循环。在后续操作中只要牵扯到这里，就会发生无限loop。这还只是多线程时可能引发的诸多问题中的一种，Java 7的`HashMap`还可能出现数据丢失等问题。

因此，Java 8把`HashMap`的插入从头插改成了尾插。那么，这样是否就能彻底解决线程不安全的问题呢？

遗憾的是，答案仍然是否定的。采用尾插法的确不会出现环形链表的情况，但是在多线程的情况下仍然不安全。下面是Java 8的`HashMap`中的`putVal`方法实现：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)  // 如果没有hash碰撞则直接插入元素
        tab[i] = newNode(hash, key, value, null);
    else {
        ...
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

注意带有注释的第6行代码。程序会在插入元素之前检测有没有发生哈希碰撞，如果没有，则会直接插入元素。考虑多线程的情形。如果线程A和线程B同时进行`put`操作，刚好这两条不同的数据哈希值一样，并且相应的桶的数据为`null`，所以线程A和线程B都会进入第6行代码中。假设线程A在判断之后、进行数据插入之前的时刻被挂起，而线程B正常执行，完成数据的插入。然后，线程A获取CPU时间片。此时问题来了：线程A进行过哈希判断了，它会直接把线程B插入的数据给覆盖掉，发生线程不安全。

## 两个因子一棵树

有两个因子直接影响`HashMap`的效率，

- 初始容量（initial capacity）。容量是指哈希表中桶的数量，而初始容量顾名思义就是`HashMap`实例创建时的最初容量。
- 载入因子（Load factor）。含义在前面已经阐述。

此外，同一桶内挤进去很多元素的时候，如果仍用链表存储，效率就会非常低下。因此，在桶内元素过多时，`HashMap`会用红黑树替换链表。

这些内容，且听下回细细分解。

## 参考资料

- [JavaDocs - HashMap](https://docs.oracle.com/javase/8/docs/api/java/util/HashMap.html)
- [掘金 - HashMap](https://juejin.im/post/5dee6f54f265da33ba5a79c8)
- [WikiPedia - Hash table](https://en.wikipedia.org/wiki/Hash_table)
- [YiKun - HashMap工作原理及实现](https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)
- [CSDN - HashMap线程不安全的体现](https://www.cnblogs.com/developer_chan/p/10450908.html)
- [CNblogs - HashMap多线程并发问题](https://www.cnblogs.com/wyq178/p/8676655.html)
- [ConcurrentHashMap & Hashtable](https://mp.weixin.qq.com/s/AixdbEiXf3KfE724kg2YIw)