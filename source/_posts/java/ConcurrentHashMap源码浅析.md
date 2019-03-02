---
title: ConcurrentHashMap源码浅析
date: 2018/2/1 17:48:25  # 文章发表时间
toc: true
tags:
- Java
categories:
- Java
- 源码阅读
thumbnail: http://pic.jingl.wang/2018-04-14-115554.jpg

---
## 基本数据结构
![](http://pic.jingl.wang/2018-04-14-114851.png)

* 用数组存储每个桶的首节点
* 使用hash值分桶
* hash值相同的元素，以链表的形式存放在桶中
* 当一个桶内的元素超过8个时，将链表转换为红黑树

<!-- more -->

## Node类
```
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
}
```
### map中最基本的结构，由4个成员变量组成：
* hash：当前结点的hash值，在子类中，hash值可以为负，有特殊含义
* key： 结点的key
* val：结点的value
* next：指向下一个结点的指针

### 此外还提供一个方法`find`方法
```
/**
 * Virtualized support for map.get(); overridden in subclasses.
 */
Node<K,V> find(int h, Object k) {
    Node<K,V> e = this;
    if (k != null) {
        do {
            //在链表中寻找
            K ek;
            if (e.hash == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
        } while ((e = e.next) != null);
    }
    return null;
}
```
当桶中元素以链表形式存储时，调用find函数寻找对应元素，当以红黑树存放时，调用`TreeNode`中的find函数查找。

## TreeBin类
当每个桶中元素超过8个时，会将链表变为一颗红黑树，TreeBin是红黑树的包装类，保存了红黑树的根节点，此外这个类提供一个读写锁，用于在红黑树进行调整的时候进行线程，红黑书的基本结点为`TreeNode`类型

## ForwardingNode类型
在扩容过程中，当前桶已经完成遍历时，将当前桶在table数组中标记为ForwardingNode类型，当调用get方法查询该桶内元素时，实际调用该类的find方法去nextTable中寻找对应数据。

## put方法
### 流程
![](http://pic.jingl.wang/2018-04-14-114947.png)
### 源码分析
```
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //当key为null时，抛出异常，所有ConcurrentHashMap的key不能为null
    if (key == null || value == null) throw new NullPointerException();
    
    //计算新节点的hash值
    int hash = spread(key.hashCode());
    int binCount = 0;
    
    //进入一个死循环，当插入成功后才能跳出循环
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh; K fk; V fv;
        
        //当前table为空，初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
            
        //对应桶首节点为空，cas插入新的桶节点
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;                   // no lock when adding to empty bin
        }
        //对应桶在进行扩容，即该结点类型为`ForwardingNode`
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        //判断首结点
        else if (onlyIfAbsent && fh == hash &&  // check first node
                 ((fk = f.key) == key || fk != null && key.equals(fk)) &&
                 (fv = f.val) != null)
            return fv;
        else {
            V oldVal = null;
            //进入临界区
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //如果以链表形式存放，将新节点插入链接最后位置
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value);
                                break;
                            }
                        }
                    }
                    //如果以红黑树形式存放，将新节点插入红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
        
            if (binCount != 0) {
            
                //如果桶内元素树达到临界值，转换为红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);、
                //如果当前key已存在，则替换对应node中的value，返回，不需要改变计数器
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```
多个线程可以同时调用，在插入时，只对对应桶进行加锁操作，粒度更细


### initTable方法
initTable可以运行多个线程同时调用，但只有一个线程可以进行初始化，其他线程通过`yield`让出cpu
### 流程图
![](http://pic.jingl.wang/2018-04-14-115015.png)
### 源码分析
```
/**
 * Initializes table, using the size recorded in sizeCtl.
 *  sizeCtl 状态：
 *  1. = -1 : 正在初始化
 *  2. < -1 : 正在扩容，数值为 -(1 + 参与扩容的线程数)
 *  3. = 0  : 创建时初始为0
 *  4. > 0  : 下一次扩容的大小
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //其他线程正进行操作，让出cpu
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
            
        /*
        * 将sizeCtl改为初始化状态，由于使用cas方法，可以保证：
        * 如果改变状态后，没有其他线程可以进入以下代码段；
        * 如果该cas方法执行失败，代表其他线程已经进入了该代码段，所以需要重新进入循环获得新的sizeCtl
        */
        else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
            try {
                //进行初始化操作
                //由于所有之前让出cpu的线程最终都会进入该代码块，所以需要在这里重新判断一下table是否为空，如果已经初始化则跳出
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

## 扩容
### 扩容的触发
在每次调用put方法进行数据插入后，都会调用`addCount`方法进行计数器的更新，当元素数量满足扩容条件后调用transfer进行扩容，多个线程可以进行协助扩容。在扩容过程中，新的数据存放在nextTable中。
### 扩容的整体思路
* 首先对nextTable数组进行初始化，该操作只能由一个数组完成。
* 对整个table数组进行遍历，将table中元素复制到nexttable中：
    * 如果当前位置元素为null，则将该位置的数组元素替换为ForwordingNode节点。
    * 如果当前位置元素为非空，则将桶内元素重新放入正确位置。
    * 如果该桶已经处理完毕，就将该桶的首元素替换为ForwardingNode节点。
* 当所有节点都完成了扩容，将table赋值为nextTable，sizeCtl复制为新容量的0.75倍
  
#### 多线程扩容
ConcurrentHashMap中存在一个叫`transferIndex`的成员变量，代表下一个需要进行元素复制的桶的下标，当存在线程对该下标对应的桶进行操作时，将transfer利用cas设置成下个需要操作的桶下标，由于使用了cas和volatile，实现了多线程进行数组遍历，进而实现多线程协同遍历;每当处理完一个节点，将该节点set成forwordingNode,当其他节点处理到这个节点后，直接跳过。
### 代码分析
```
/**
 * Moves and/or copies the nodes in each bin to new table. See
 * above for explanation.
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range

    //当nextTable==null，初始化nextTable
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            //将nextTable初始化为table大小的2倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        //设置transfetIndex为table最后一个桶
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    
    //开始循环遍历（table从后往前）
    //i为当前进行复制的桶，bound表示当前线程需要复制下标在bound之前的桶
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        
        //遍历从nextIndex到transferIndex之间的桶
        while (advance) {
            int nextIndex, nextBound;
            
            //当前桶还在bound之间
            if (--i >= bound || finishing)
                advance = false;
            
            //当table中所有的桶已经被复制
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //当线程首次进入该循环，
            //或已经完成上次设置范围内桶复制，但还有桶未被复制时
            //设置该线程需要复制的桶的下标范围
            else if (U.compareAndSetInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        
        //完成扩容
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //当finishing标记为ture时，表示整个扩容过程已经完成，进行扫尾工作
            if (finishing) {
                //将table 赋值为 nextTable； 将nextTable赋值为null 重置 sizeCtl
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            
            //该线程已经完成了扩容任务，在退出前需要检查其他线程是否还在进行扩容
            //如果还存在未完成的线程，结束操作
            //如果所有线程已经完成扩容，则将finishing标记为true，在进行一次完成扩容的检查。
            if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //判断其他线程是否还在进行扩容
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                    
                //所有线程都完成扩容
                finishing = advance = true;
                
                //由于在退出之前需要再次对所有节点进行一次检查，所有将i再次设置为n，对table再次进行一次遍历
                i = n; // recheck before commit
            }
        }
        //如果当前桶中没有元素，修改为forwardingNode
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //当前桶已经完成扩容，跳过该桶，进入下一个桶
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        //没有进行过扩容操作，将元素从table中移动到nextTable中
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```






















