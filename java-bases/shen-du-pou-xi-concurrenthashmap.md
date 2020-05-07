# 深度剖析ConcurrentHashMap

## 1.原理解析

线程安全的保证：CAS+Synchonized 数据存储实现：数组+链表+红黑树

![ConcurrentHashMap &#x7684;&#x5B58;&#x50A8;&#x7ED3;&#x6784;](../.gitbook/assets/image%20%281%29.png)

### 1.1 成员变量

* table： `transient volatile Node<K,V>[] table` 一个Node类型的表，默认为null，初始化在第一次put操作时，默认大小为16，扩容时大小总是2的幂次方
* nextTable： `private transient volatile Node<K,V>[] nextTable` 默认为null，扩容时新生成的数组，大小为原数组的2倍
* sizeCtl: `private transient volatile int sizeCtl` 默认为0，用来控制table的初始化和扩容操作。 -1 表示正在table初始化，-N表示有N-1个线程正在扩容 如果table未完成初始化，则表示table初始化需要的大小，如果已经完成初始化，则表示table的容量，默认为table大小的0.75倍
* Node： 保存key和value以及key的hash值的数据结构。其中value和next都用volatile修饰，保证了并发中的可见性。

```text
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
}
```

ForwardingNode： 一个特殊的节点，hash值为-1，只有在扩容的时候使用，作为一个占位符，表示当前节点为null或者已经迁移。

```text
static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }
}
```

## 2.常见方法

### 2.1 初始化table

ConcurrentHashTable的初始化操作是在第一次put操作时进行的，而且只会初始化一次：

![putVal&#x64CD;&#x4F5C;](../.gitbook/assets/image%20%282%29.png)

这里的initTable，就是对table变量进行初始化操作：

```text
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)//如果一个线程发现当前sizeCtl小于0，表明当前有其他
                              //线程在对table执行cas成功，需要当前线程让出时间片
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
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

这里讲解下`U.compareAndSwapInt(this, SIZECTL, sc, -1)`操作哈  
这里的U是`private static final sun.misc.Unsafe U;`，是[Unsafe类](https://www.jianshu.com/p/db8dce09232d)，这个方法CompareAnsSwapInt，就是乐观锁CAS了。这里有四个参数，第一个，第二个参数用来确定当前操作对象在内存中的存储值，然后和第三个expect value比较，如果相等，则将内存值更新为第四个updaet value值。关于CAS的介绍，在[这里](https://www.jianshu.com/p/c42770f3e4bb)详细介绍。在原子性的保证下，将sc的值设置为-1，表明当前table正在初始化。

### 2.2 put操作

直接看源码，参看源码进行解释：

```text
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
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
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
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
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
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

putVal的操作可以分为三个步骤【假设table已经初始化了】：

* bucket为空时，采用CAS将Node放到对应的bucket中
* 当前map正在扩容`f.hash == MOVED`，则先进行扩容，再进行更新值
* 发生hash冲突时，先使用synchonized字锁住。倘若当前的hash值对应的是**链表**的头节点，则遍历链表，如果能找到hash对应的节点，则更改对应的值，否则在链表的尾部增加节点；倘若当前的hash值对应的是**红黑树**的根节点，则在树结构上遍历，进行更新或者插入操作。 整个putval的操作中，是用for循环来外包的。当遇到map正在扩容时，没有break操作，会等待map扩容结束后，进行更新或者插入值。 这里一些细节点再进行介绍下：

#### 2.2.1 hash算法

这里和hashmap一样：

```text
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```

#### 2.2.2 获取table所对应的索引元素

```text
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
```

采用`Unsafe.getObjectVolatie()`来获取，而不是直接用table\[index\]的原因跟ConcurrentHashMap的弱一致性有关。在java内存模型中，我们已经知道每个线程都有一个工作内存，里面存储着table的副本，虽然table是volatile修饰的，但不能保证线程每次都拿到table中的最新元素，`Unsafe.getObjectVolatile`可以直接获取指定内存的数据，保证了每次拿到数据都是最新的。

### 2.3 扩容

为什么会扩容？

* 当ConcurrentHashMap的tab长度大于64时，会使用红黑树
* 新增节点后，如果链表的长度大于8时，会调用`treeifyBin`把链表转换为红黑树。在转换结构时，如果tab的长度小于`MIN_TREEIFY_CAPACITY`，默认是64，则会将数组长度扩大到原来的两倍。并触发`tranfer`，重新调整节点的位置
* 新增节点后，如果`addCount`中统计的节点数超过`sizeCtl`，也会触发`tranfer`，进行位置调整

#### 2.3.1 addCount

```text
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```

每次put后，会计算节点的数量

#### 2.3.2 treeifyBin

```text
private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```

将链表转换为红黑树

#### 2.3.3 transfer

```text
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
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

扩容的代码有点长，简单的过程描述是，当tab元素的量达到容量阈值sizeCtl时，触发扩容：

1. 构建一个NextTable，其大小为Table的两倍
2. 把table中的数据，复制到NextTable中 在扩容的过程中，依然支持并发更新操作，也支持并发插入。 _如何在扩容时，并发地复制与插入？_

* 遍历整个table，当前节点为空，则采用CAS的方式在当前位置放入fwd
* 当前节点已经为fwd\(with hash field “MOVED”\)，则已经有有线程处理完了了，直接跳过 ，这里是控制并发扩容的核心
* 当前节点为链表节点或红黑树，重新计算链表节点的hash值，移动到nextTable相应的位置（构建了一个反序链表和顺序链表，分别放置在i和i+n的位置上）。移动完成后，用Unsafe.putObjectVolatile在tab的原位置赋为为fwd, 表示当前节点已经完成扩容。

### 2.4 get 读操作

```text
public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

读操作比较简单了，不需要控制并发，如果tab为空，返回null，否则计算hash值，找到bucket中对应的位置，如果是node节点直接返回，否则，返回null。

## 3.在jdk1.7和1.8之间的不同

### 3.1 并发机制

* 在jdk1.7中，concurrentHashMap是通过锁分段技术。把整个table分为几个segment，每个segment中分配一个锁，每个segment下，又分table+HashEntry。通过给segment加一个重入锁，实现线程安全。【相比于hashtable，就是把hashtable分割为多个段，细化锁的粒度】。
* 在jdk1.8中，直接使用CAS + synchronized保证并发更新。粒度更细，不会在一个segment上加一把锁了。

### 3.2 put操作

* 在1.7中，多个线程同时竞争获取同一个segment锁，获取成功的线程更新map；失败的线程尝试多次获取锁仍未成功，则挂起线程，等待释放锁
* 在1.8中，访问相应的bucket时，使用sychronizeded关键字，防止多个线程同时操作同一个bucket，如果该节点的hash不小于0，则遍历链表更新节点或插入新节点；如果该节点是TreeBin类型的节点，说明是红黑树结构，则通过putTreeVal方法往红黑树中插入节点；更新了节点数量，还要考虑扩容和链表转红黑树

## 4 ConcurrentHashMap能替换HashTable吗？

hash table虽然性能上不如ConcurrentHashMap，但并不能完全被取代，两者的迭代器的一致性不同的，hash table的迭代器是强一致性的，而concurrenthashmap是弱一致的。 ConcurrentHashMap的get，clear，iterator 都是弱一致性的。  
下面是大白话的解释：

* Hashtable的任何操作都会把整个表锁住，是阻塞的。好处是总能获取最实时的更新，比如说线程A调用putAll写入大量数据，期间线程B调用get，线程B就会被阻塞，直到线程A完成putAll，因此线程B肯定能获取到线程A写入的完整数据。坏处是所有调用都要排队，效率较低。
* ConcurrentHashMap 是设计为非阻塞的。在更新时会局部锁住某部分数据，但不会把整个表都锁住。同步读取操作则是完全非阻塞的。好处是在保证合理的同步前提下，效率很高。坏处 是严格来说读取操作不能保证反映最近的更新。例如线程A调用putAll写入大量数据，期间线程B调用get，则只能get到目前为止已经顺利插入的部分 数据。

选择哪一个，是在性能与数据一致性之间权衡。ConcurrentHashMap适用于追求性能的场景，大多数线程都只做insert/delete操作，对读取数据的一致性要求较低。

Reference：  
\[1\] [https://blog.csdn.net/programmer\_at/article/details/79715177](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fprogrammer_at%2Farticle%2Fdetails%2F79715177)

