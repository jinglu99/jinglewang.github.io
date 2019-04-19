---
title: HashMap源码
date: 2019/04/19 14:00:00  # 文章发表时间
toc: true
tags:
- Java
categories:
- Java
- 源码阅读
thumbnail: http://pic.jingl.wang/2019-04-19-u-1418578175-1769712762%26fm-26%26gp-0.jpg
---
HashMap是一个基于散列表的Map实现，和HashTable类似，实现了所有的Map方法，但与HashTable不同的是，HashMap允许key和value的值为null，且HashMap是线程不安全的。HashMap不会保证映射的顺序。
<!-- more -->

HashMap内部会维护一个数组（table），数组中的每一个元素作为一个桶，插入新元素时，会通过key的hashCode 做hash计算，来知道新元素会被分配到哪个桶中，然后将其插入对应的桶。一个桶的结构有时是一个链表，有时是一棵红黑树，这取决于桶中元素的个数。

![](http://pic.jingl.wang/2019-04-18-072736.png)

对于一个Hash表来说，表中的元素最理想的状态是分布均匀的（最好每个桶中只有一个元素），如果太紧，则会影响查询性能，如果太松散，会增加占用的内存空间，所以需要一定的规则来控制table的大小，在HashMap中，有2个初始化参数来影响HashMap的性能，初始容量和负载因子，初始容量是初始化HashMap时table的长度，负载因子控制触发HashMap扩容时的阈值，一般负载因子为0.75。

HashMap元素的迭代是Fail-fast的，如在迭代时，发生结构修改，那么迭代器会抛出`ConcurrentModificationException`异常。

## 构造方法

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

HashMap提供了四个构造方法，默认的构造方法，将负载因子赋值为0.75，此外，还提供了可以自定义初始化容量和负载因子的构造函数，最后一个构造函数用于复制一个已经存在的Map。

在第一个构造方法中，有一个`tableSizeFor`方法，用来计算离initialCapacity最近的一个2次幂，详细分析可以看[tableSizeFor分析](<https://blog.csdn.net/fan2012huan/article/details/51097331>)。

## Put

```java
	/**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;	//初始化table
        if ((p = tab[i = (n - 1) & hash]) == null)
            //如果桶中没有元素那么直接将新元素置为桶的第一个元素。
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                //桶中第一个元素即为目标位置
                e = p;
            else if (p instanceof TreeNode)
                //如果当前桶是一棵，红黑树，则忘树中插入新元素
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //如果当前桶是一个连表结构的，在连表中寻找是否已经存在key
                //如果有则替换为新value
                //如果没有则在尾部添加新元素
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //插入至链表尾部
                        p.next = newNode(hash, key, value, null);
                        //当桶中元素数超过TREEIFY_THRESHOLD时 将连表转化为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);		//子类可重新该方法
                return oldValue;
            }
        }
        ++modCount;	//记录当前HashMap被修改的次数，用于在迭代时判断HashMap是否被修改，保证Fail-fast特性
        if (++size > threshold)
            resize();	//扩容
        afterNodeInsertion(evict);	//回调方法 子类重写
        return null;
    }



 	static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

put时，HashMap会重新计算key的hash值，hash()方法通过对原hashCode的高16位和低16位做了异或处理， 减少hash碰撞，详细的分析见[hash](<https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/>)。

第一次插入时，会调用resize()方法来初始化table，这一步的分析在之后展开，当初始化table完毕后将会进入真正的插入流程。

首先，put方法会通过key的哈希值，计算当前key所在的桶的序号。

```
(n - 1) & hash
```

这时已经知道了这个key应该放到哪个桶中了，这时会遇到三中情况：

* 桶中还没有元素及table[(n - 1) & hash] == null
* 桶中是链表结构
* 桶中是红黑树结构

当桶中还没有元素时，直接将table[(n - 1) & hash]赋值为新元素，如果桶是红黑树结构的，那么就将新元素插入红黑树中，如果是链表结构的，则会将新元素插入至链表的尾部，且此时，会进行一个判断，如果桶中元素个数超过了TREEIFY_THRESHOLD，且table数组长度超过64，则会将这个链表转化为一颗红黑树。

_modCount是用来记录当前HashMap被修改结构（改变map中映射个数的修改）的次数，HashMap以此来保证HashMap迭代的`Fail-fast`机制, 在使用迭代器时，会用这个字段来判断在迭代过程中，HashMap是否有被插入或删除过新值，如果有则会抛出异常。_

### 扩容

HashMap中数据均匀的分布是保证HashMap性能的最核心的要求，之前说过，HashMap是有许多的桶构成的，每一个键值对都放在相应的桶中，在查询时需要先找到存放的桶，然后再在桶中寻找对应的key，桶在HashMap中是一数组的形式存在的，所以只需要知道桶的下标即可，这一个步骤的时间复杂度为O(1), 在桶中寻找key，这一部分的时间复杂度为O(n) (链表) 或者 O(logn)(红黑树)。 因此如果桶的数量太少的话，那么势必每个桶中的键值对将会增加，在桶里寻找key的这个过程的性能将会降低，所以，需要增加桶的数量，将原来桶中的键值对，再重新分配到新的桶中，这就是HashMap的扩容。

HashMap的扩容，是通过resize()方法实现的。

```java
	final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                //当table大小已经超过了最大的容量
               	//将threshold置为最大值
                //以后将不再会触发扩容操作
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //将table长度乘2
                //threshold*2
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            //初始化
            //capacity默认为8
            //threshold默认为 8 * 0.75
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            //初始化时计算threshold
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            //将原来桶中的映射放入新桶中
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
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

HashMap中，table的长度必须为2的n次幂，这是由HashMap的hash算法所决定的，在之前提到过，计算桶的下标的方法是这样的：

```
(n - 1) & hash
```

n是table的长度，举个例子，如果table的capacity是4，那么4-1的二进制就是 0011，与hash做&运算，结果也必然是在0-3之间的。

那么为什么一定是2的n次幂呢？

2的n次方实际就是1后面n个0，2的n次方-1  实际就是n个1； 例如长度为9时候，3&(9-1)=0  2&(9-1)=0 ，都在0上，碰撞了； 例如长度为8时候，3&(8-1)=3  2&(8-1)=2 ，不同位置上，不碰撞；

这样可以减少hash碰撞的概率。

### 转换红黑树

```java
	final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```

将链表转换为红黑树的操作使用treeifyBin实现的，构造一颗红黑树的具体实现不做详细分析，在上述代码中，我们可以看到触发这个转换的条件除了桶中的元素必须大于8个之外，还需要满足table的长度大于64，如果不满足，则只是触发扩容操作。

## get

经过对put的分析，get的实现逻辑已经很清晰了。

```java
	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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

首先通过hashCode计算桶下标，在对应桶里面寻找对应的key，如果是链表则遍历，如果是红黑树，那么就以二叉查找数的方法查询。



## 遍历

HashMap提供了三个用于遍历的方法：

* keySet()
* values()
* entrySet()

这三个方法分别返回了一个AbstractSet的子实现类，对于set的遍历，我们需要看一下他们迭代器的实现。

```java
//keySet
public final Iterator<K> iterator()     { return new KeyIterator(); }

//ValueSet
public final Iterator<V> iterator()     { return new ValueIterator(); }

//EntrySet
public final Iterator<Map.Entry<K,V>> iterator() { return new EntryIterator(); }
```

由源码看出，这三个set的迭代器的实现类分别是：

* KeyIterator
* ValueIterator
* EntryIterator

这三个类有一个共同的父类`HashIterator`

```java
 	abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot

        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }

    final class KeyIterator extends HashIterator
        implements Iterator<K> {
        public final K next() { return nextNode().key; }
    }

    final class ValueIterator extends HashIterator
        implements Iterator<V> {
        public final V next() { return nextNode().value; }
    }

    final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
        public final Map.Entry<K,V> next() { return nextNode(); }
    }
```

由以上代码可以看出，这三个迭代器的实现类只重写了next方法，分别返回key， value 和 entry，主要的实现都放在了HashIterator中。

在创建一个迭代器时，首先会记录此时的modCount， 之前说过，这个字段是用来判断在迭代过程中HashMap是否被修改的，index是当前所在的桶的序号，然后将迭代器的头个元素指向第一个Node。

在三个重写的next()方法中，都会调用nextNode()方法，这是迭代器的获得下一个元素的实现，他会依次遍历HashMap中的每一个桶，在每次调用时，都会比较当前modCount和之前记录的modCount是否相同，如果不同，则代表HashMap已经发生了结构修改，那么此时就会抛出`ConcurrentModificationException`异常，这就是HashMap遍历时的Fail-fast机制。





