---
title: 玩一玩ConcurrentHashMap源码
toc: true
date: 2018-08-24 15:04:33
tags: [ConcurrentHashMap,红黑树]
---
## 前言
很早之前就想自己写一篇总结，看了许多优秀文章的讲解，但源码这种东西很久不去看就容易忘，虽然下次再去看，领悟超快。前篇[《聊一聊Java并发》](http://crazysunj.com/2018/08/03/%E8%81%8A%E4%B8%80%E8%81%8AJava%E5%B9%B6%E5%8F%91/)文末说到要写一篇介绍ConcurrentHashMap的，吹出去的牛，只能硬抗了。一般分析ConcurrentHashMap我们都分为JDK1.7和1.8去讲解，毕竟数据结构和处理并发的方式都变了。
## 源码分析
本篇文章主要针对重要的数据结构、构造函数、put、get和size方法进行讲解。对象又分为JDK1.7和JDK1.8。
<!--more-->
### JDK1.7
#### 数据结构

```
//ConcurrentHashMap
/**
 * 采用分离锁技术，最多分成16段
 */
final Segment<K,V>[] segments;

static final class Segment<K,V> extends ReentrantLock implements Serializable {
    ...
    /**
     * 每个Segment的Hash表，用于存储数据
     */
    transient volatile HashEntry<K,V>[] table;

    /**
     * 每个Segment中元素的数量
     */
    transient int count;

    /**
     * 扩容阈值，受loadFactor影响
     */
    transient int threshold;

    /**
     * 负载因子，确定threshold
     */
    final float loadFactor;
    ...
}

static final class HashEntry<K,V> {
    /**
     * key的hash值
     */
    final int hash;
    final K key;
    volatile V value;
    /**
     * 链表结构
     */
    volatile HashEntry<K,V> next;
    ...
}
```
综上所述，可把ConcurrentHashMap的数据结构归纳为：

![](/img/concurrenthashMap_1_7_structure.png)

#### 构造函数

```
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
    ...
    // concurrencyLevel最大为2的16次幂，默认是16
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // segmentShift是hash的偏移位，用32减是因为hash是32位
    this.segmentShift = 32 - sshift;
    // segmentMask可以认为是掩码，用于取高位的数值
    this.segmentMask = ssize - 1;
    // initialCapacity是初始容量，最大为2的30次幂，默认是16
    // 用于分配Segment的容量
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    // cap是Segment的容量，最小值是2
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // loadFactor负载因子，默认是0.75，用于Segment
    // 创建第一个Segment，方便后面取各种参数
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
     // 创建Segment数组，一旦创建，不允许扩容，添加第一个Segment
     // 假设全部都是默认值，那么(int)(cap * loadFactor)是1，也就是说再添一个就得扩容- -!
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0);
    this.segments = ss;
}
```

#### PUT

```
public V put(K key, V value) {
    Segment<K,V> s;
    ...
    int hash = hash(key);
    // 取高位定为j，用于计算Segment的偏移地址，可以认为是下标
    int j = (hash >>> segmentShift) & segmentMask;
    /**
     * Class sc = Segment[].class
     * SBASE = UNSAFE.arrayBaseOffset(sc)，SBASE是Segment数组的偏移地址
     * ss = UNSAFE.arrayIndexScale(sc)，ss表示Segment数组的大小
     * SSHIFT = 31 - Integer.numberOfLeadingZeros(ss)，SSHIFT可以认为是ss最高位为1的离最低位的距离
     * 例如ss为1010，那么SSHIFT就为3
     * (j << SSHIFT) + SBASE就是当前存储的Segment在segments的偏移地址
     * 如果segments获取到的Segment为空，说明还没创建，那么新创建一个，否则Segment把数据put进去
     */
    if ((s = (Segment<K,V>)UNSAFE.getObject
         (segments, (j << SSHIFT) + SBASE)) == null) 
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```

再来看看如何新创建Segment的。

```
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    // u是待创建的Segment在segments的偏移地址
    long u = (k << SSHIFT) + SBASE; 
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        // 利用构造函数中初创的Segment获取各种属性
        Segment<K,V> proto = ss[0]; 
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { 
            // 创建新的Segment
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            // CAS自旋常规操作把新创建的Segment刷新到segments数组中，刷新成功把新创建的返回
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

如果对CAS不太了解的可以看看[《聊一聊Java并发》](http://crazysunj.com/2018/08/03/%E8%81%8A%E4%B8%80%E8%81%8AJava%E5%B9%B6%E5%8F%91/)。然后把数据添加到Segment中，

```
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 如果看的比较仔细知道Segment继承ReentrantLock，故可调用tryLock尝试获取锁，假设没拿到锁，那么将自旋获取锁，下面讲解scanAndLockForPut方法
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // 可以认为index是table数组的下标
        int index = (tab.length - 1) & hash;
        // 基于Unsafe获取table下标为index的HashEntry
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    // 如果引用相等，或者equals相等，说明已经找对key了，修改对应value就行了（value是volatile修饰），这里也说明了为什么在写类的时候要实现equals方法
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                    	// value由volatile修饰
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                // 如果没找到，继续下一个
                e = e.next;
            }
            else {
                // 如果下标下的值为空，说明还没填进去，那么就得新创建一个
                // 无论什么情况，node都成为链表表头，原来的表头都在其后面
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                // 已创建节点，那么长度+1
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    // 扩容
                    rehash(node);
                else
                    // 把新建节点插入到table数组中，同样基于Unsafe
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 释放锁
        unlock();
    }
    return oldValue;
}
```

这里可能需要解释的是scanAndLockForPut方法，因为该方法创建了新的节点并存储了值，而后面的扩容方法rehash就不讲解了，没啥意义，无非是扩大数组长度，重新放入新的容器。这里简单总结下put的流程，首先找到要put的对象在table的下标，其次取出遍历，若找到对应key直接修改值，反之，创建新节点存储新的key和value并插入表头，若插入容量不够就扩容。

OK，现在我们来讲解下scanAndLockForPut方法。

```
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    // 这里的first相当于上面put方法中first
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    // 用于记录自旋尝试获取锁次数
    int retries = -1; 
    while (!tryLock()) {
        HashEntry<K,V> f; 
        if (retries < 0) {
            // retries为初始值或者被重置
            if (e == null) {
                // 一直找到最后都没有找到对应key，那么就创建新的节点并存储key和value，retries置为0
                if (node == null) 
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
                // 找到了对应key，retries直接置为0
                retries = 0;
            else
                // 找不到往下走直至找到key或者为null
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {
            // 达到自旋锁最大次数，阻塞等待获取锁，相当于重量级锁synchronized
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
            // 自旋过程中，如果链表头被改变，那么将会被重置，重新来过
            e = first = f; 
            retries = -1;
        }
    }
    return node;
}
```

scanAndLockForPut方法主要目的不是创建一个节点，而是获取锁，如果在获取锁的过程中又创建了节点或者找到了key那固然最好。那么到此，put操作就分析完了。
#### GET

```
public V get(Object key) {
    Segment<K,V> s; 
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    // 根据偏移地址获取对应的Segment
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        // 获取Segment中table对应下标的链表头HashEntry，开始遍历查找对应key
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                // 找到直接返回
                return e.value;
        }
    }
    return null;
}
```

有了put的分析经验，get相对较为容易。

#### SIZE

```
public int size() {
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; 
    long sum;        
    long last = 0L;   
    int retries = -1; 
    try {
        for (;;) {
            if (retries++ == RETRIES_BEFORE_LOCK) {
                // 如果尝试三次之后仍然无法获取正确的size，那么所有Segment将全部锁住以保证获取正确的size，事不过三哈，老外也很6
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); 
            }
            sum = 0L;
            size = 0;
            overflow = false;
            // 遍历所有Segment，用size叠加每个Segment中的count来统计整个Map的size
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        // 为什么会小于0？因为int的最大值是32位且有个具体值是Integer.MAX_VALUE，如果再加1遍溢出小于0了
                        overflow = true;
                }
            }
            // 如果前一次操作和本次操作的操作统计数不变，说明在我们统计size的时候，并没有其它线程在操作，说明是正确的，停止循环
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        // 如果进行了加锁操作固然要解锁
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    // 这里返回最终的size，如果溢出是返回int的最大值
    return overflow ? Integer.MAX_VALUE : size;
}
```

#### 总结
我只能从结构去理论分析，毕竟自己并没有从事高并发工作，大家就当吹吹牛逼了。1.7的ConcurrentHashMap采用分段锁的技术，可以保证在一定程度上一个操作不会阻塞另一个操作，每一段都是一把锁，线程安全。采用volatile+CAS提高效率，其中get方法最为体现。但缺点也很明显，因为分段而增加了多余的时间复杂度，效率应该是偏低的。其中还有一个致命的缺点就是ConcurrentHashMap属于弱一致性，这是没办法的，鱼与熊掌怎可兼得？一致性和效率总要有一个权衡。

那什么是弱一致性？就拿ConcurrentHashMap来说，get方法看似很高效，利用了volatile，但其实有问题的，你想想如果在我们获取value之前，其它线程进行了clear或者说remove，那么我们获取的数据其实是无效的。但是你再get一遍，或许就是正确的，所以会有CAS操作。

### JDK1.8
讲道理，JDK1.8相对JDK1.7的改动还是很大的，毕竟大体的结构上得跟着HashMap的步伐走，所以在分析的时候常常要单独分析。废话不多说，一起来看吧。
#### 数据结构

```
// 用于存储数据，相当于JDK1.7的HashEntry
transient volatile Node<K,V>[] table;

// 节点计数
private transient volatile long baseCount;

/**
 * 用于控制ConcurrentHashMap的大小
 * -1标识初始化
 * 0标识默认值
 * 小于-1标识多个线程在扩容
 * 大于0表示阈值或者容量
 */
private transient volatile int sizeCtl;

// 惊人的一致
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
    ...
}

// JDK1.8之后优化链表过长而采用红黑树
static final class TreeNode<K,V> extends Node<K,V> {
    // parent,left,right二叉树标准结构
    TreeNode<K,V> parent;  
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    // 便于删除操作
    TreeNode<K,V> prev;
    // 是否为红    
    boolean red;
}

// 用于代理TreeNode
static final class TreeBin<K,V> extends Node<K,V> {
    // 根节点
    TreeNode<K,V> root;
    // 头节点
    volatile TreeNode<K,V> first;
    // 最近一个WAITER标识的线程
    volatile Thread waiter;
    // 锁状态
    volatile int lockState;
    // 标识位
    static final int WRITER = 1; // 写
    static final int WAITER = 2; // 等待
    static final int READER = 4; // 读
}
```

大体的结构是这样的：

![](/img/concurrenthashMap_1_8_structure.png)

#### 构造函数

```
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
    ...
    // 不再限制concurrencyLevel，其实是concurrencyLevel和loadFactor已经失去作用，可以认为是为了兼容老版本
    if (initialCapacity < concurrencyLevel)
        initialCapacity = concurrencyLevel; 
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

真TM简洁。连初始化table都没做。

#### PUT

```
public V put(K key, V value) {
    return putVal(key, value, false);
}

// 长到不想看
final V putVal(K key, V value, boolean onlyIfAbsent) {
    ...
    // 算出key的hash
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            // 如果table未初始化便初始化，由构造函数知道，懒加载
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 根据hash计算偏移地址并判空，若为空，CAS插入新节点，若插入成功，循环结束，否则下次循环继续插
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break; 
        }
        else if ((fh = f.hash) == MOVED)
            // 如果是ForwardingNode，即正在扩容，那么进行扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 亲切的synchronized又回来了
            synchronized (f) {
                // 再一次确定偏移地址i的值是否变动
                if (tabAt(tab, i) == f) {
                    /**
                     * 若不变动，判断fh
                     * 大于0当做普通链表处理
                     * -1是ForwardingNode
                     * -2是TreeBin
                     */
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            // 遍历节点，如果找到对应key的节点修改其值，遍历到最后还未找到那么在最后插入保存key和value的新节点
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
                        // 红黑树插入
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            // 如果不为空，说明找到了，那么修改值即可
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
                if (binCount >= TREEIFY_THRESHOLD)
                    如果插入节点后大于等于链表转红黑树的阈值(8)，那么进行转换
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 判断要不要扩容
    addCount(1L, binCount);
    return null;
}
```

有种想放弃的冲动有木有？看到想睡觉有木有？这个方法里面还有几个重量级方法都没分析，让暴风雪来得更猛烈些吧！我们按顺序来吧，如何初始化table的。

```
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            // 正在扩容或者初始化状态，线程挂起
            Thread.yield(); 
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // 初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 其实就是0.75n，可联想我们的负载因子
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

接下来再分析，如果正在扩容，然后进行帮助扩容操作。

```
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 如果不符合，应该是扩容结束了
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        // 计算了一串牛逼的数字
        int rs = resizeStamp(tab.length);
        // 循环判断是否扩容结束
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                // 应该是扩容结束了，直接跳出循环
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                // 帮助扩容，sc+1
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

重头戏来了，那就是扩容（迁移）。主要难点是有链表和红黑树两种结构。前方高能，实在太长了。

```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    // stride，每个任务负责迁移的hash桶
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; 
    if (nextTab == null) {            
        try {
            // 扩容一倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      
            // 一般来说，内存溢出了
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        // 利用transferIndex开始分配任务
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 正在扩容的节点被封装成ForwardingNode
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // advance标识是否处理
    boolean advance = true;
    // finishing标识扩容是否结束
    boolean finishing = false; 
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                // 缩减区间，本次结束或者全部结束
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                // 分配任务结束
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                // 分配到任务，设置任务区间，可以知道是从后往前分配
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            /**
             * i<0说明自己任务结束
             * i>=n应该是异常情况吧，跟预期不一样
             * i+n>=nextn，正常情况，nextn应该为2n，那么其实跟i>=n一样
             */
            int sc;
            if (finishing) {
                // 扩容结束
                nextTable = null;
                // 赋值
                table = nextTab;
                // 阈值为1.5n，原来为0.75n
                sizeCtl = (n << 1) - (n >>> 1);
                // 退出循环
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 表示参与的任务结束了，可以参考上面的helpTransfer方法，如果帮助扩容，那么会+1
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    // 表示自己参与的任务完成了
                    return;
                // 表示所有任务结束了
                finishing = advance = true;
                i = n; 
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            // 如果迁移的节点是null，那么插入新生成的ForwardingNode，可以认为是占位节点
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            // 已经处理过了，直接过
            advance = true; 
        else {
            synchronized (f) {
                // 有点类似put方法的逻辑，再次确认是否一致，很重要
                if (tabAt(tab, i) == f) {
                    // 分成两类，看见l和h我总想到low和high，但并不对，其实是分成hash在x位是0和1两类，其中x为以2为底n的对数
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            // 遍历得到最后一个不一样的runBit，并用lastRun记录其节点
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // ln记录x位是0的，hn记录x位是1的
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
                            // 遍历链表，根据x位是0或者1分成两个链表，可以认为是倒序，但不完全，例如lastRun及以后还是正常顺序
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // x位是0的插入新数组i的位置，x位是1的插入新数组的i+n位置，老数组的i位置修改为新建的ForwardingNode节点
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        // 记录该位置已经处理过
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        // lo记录是0的头节点，loTail可以认为是临时节点，用于记录的，下面的hi和hiTail也一样
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        // lc记录是0的节点个数，hc记录是1的节点个数
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            // 红黑树跟链表操作一样，同样根据0和1分成两个链表
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
                        /**
                         * 判断统计变量lc是否大于UNTREEIFY_THRESHOLD(默认是6)
                         * 如果大于说明需要红黑树
                         * 再判断hc是否等于0，如果等于说明x位是1的并没有，那么红黑树节点就是原来的t
                         * 下面的判断hc同理
                         */
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        // 跟链表操作一样
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

算是把扩容解释完了！那么剩下比较关心的是红黑树是如何插入一个新节点的，其次链表是如何转换红黑树的。解答在扩展小节。
#### GET

```
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 先判断首节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 再判断非链表节点
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 循环查找对应key
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

依然很简洁！
#### SIZE

```
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}

static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```
这里，其实我漏讲了addCount方法，baseCount在讲数据结构的时候已经解释了，用来统计元素数量，那么为啥还要加入CounterCell呢？因为baseCount的更新是基于CAS操作的，那么可能会更新失败，那么CounterCell就是用于记录失败的次数，其value值一般是1或者-1，而这些操作都发生在addCount方法。
#### 总结
不再使用分段锁，不再使用ReentrantLock，而是采用了synchronized，先是JDK1.6对其的优化，然后JDK1.8的回归说明效率有很大的提升。JDK1.8HashMap采用了链表+红黑树。
### 扩展
红黑树是每个节点都带有颜色属性的二叉查找树，颜色为红色或黑色。在二叉查找树强制一般要求以外，对于任何有效的红黑树我们增加了如下的额外要求：

* 节点是红色或黑色。
* 根是黑色。
* 所有叶子都是黑色（叶子是NIL节点）。
* 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
* 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

示例图：

![](/img/red_black_tree.png)

当规则被破坏的时候，一般采用变色和旋转进行调整，具体参照维基百科，上面也是完美复制维基百科的话。

我们再来看看ConcurrentHashMap中红黑树，我们只要看链表转红黑树就行了，因为转换相当于一个插入的过程，那么基本逻辑都在。

```
TreeBin(TreeNode<K,V> b) {
    super(TREEBIN, null, null, null);
    this.first = b;
    TreeNode<K,V> r = null;
    for (TreeNode<K,V> x = b, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (r == null) {
            // 第一个节点即根节点为黑色
            x.parent = null;
            x.red = false;
            r = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = r;;) {
                int dir, ph;
                K pk = p.key;
                // 判断该节点在左还是右，小于等于0在左，反之在右，这里基本走最后一个判断分支，因为hash肯定一样
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    // 先用Comparable判断正负，如果等于0采用System.identityHashCode继续判断
                    dir = tieBreakOrder(k, pk);
                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    // 插入到红黑树中
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    // 调整红黑树
                    r = balanceInsertion(r, x);
                    break;
                }
            }
        }
    }
    this.root = r;
    assert checkInvariants(root);
}

static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
    // 初始化为红色
    x.red = true;
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        if ((xp = x.parent) == null) {
            // 父节点为空，说明x为根节点，设为黑色
            x.red = false;
            return x;
        }
        else if (!xp.red || (xpp = xp.parent) == null)
            // 如果父节点为黑色或者祖父节点为空，直接结束，调整完毕
            return root;
        if (xp == (xppl = xpp.left)) {
            if ((xppr = xpp.right) != null && xppr.red) {
                // 父节点是祖父节点的左孩子，且祖父的右孩子(叔父)为红色
                // 叔父节点、父节点设为黑色，祖父节点设为红色
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                // 向上调整
                x = xpp;
            }
            else {
                // 父节点是祖父节点的左孩子，祖父节点无右孩子
                if (x == xp.right) {
                    // 调整节点是父节点的右孩子，右旋
                    root = rotateLeft(root, x = xp);
                    // x变为父节点，调整其父和祖父节点
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    // 父节点设为黑色
                    xp.red = false;
                    if (xpp != null) {
                        // 祖父节点设为红色（颜色互换），右旋
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
        else {
            // 父节点是祖父节点的右孩子
            if (xppl != null && xppl.red) {
                // 叔父节点为红色，那么叔父和父节点均改为黑色，祖父节点改为红色
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                // 向上调整
                x = xpp;
            }
            else {
                if (x == xp.left) {
                    // x是父节点的左孩子，右旋，x变为父节点
                    root = rotateRight(root, x = xp);
                    // 调整父和祖父节点
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    // 父节点设为黑色
                    xp.red = false;
                    if (xpp != null) {
                        // 祖父节点设为红色（颜色互换），左旋
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
```

红黑树的介绍到此为止。如果对上面的代码逻辑有疑问的同学可以自己画几种情况去尝试走逻辑，虽然白费功夫，但是是值得的，要一直记得红黑树的5条规则哦。

## 终极总结
为什么HashMap是线程不安全的呢？为什么不用HashTable呢？

第一个问题，举个大家都在举的例子且我们文中也分析到了，就是在扩容的时候，其实会倒序，可以说是头插法。那么在并发的时候，一个正一个反很大概率形成环（退出循环条件是判空），最后进入死循环。

第二问题，HashTable优点就是强一致性，但也是致命的，如果高并发的情况，那么将造成大量阻塞，效率极低。适用于绝对的线程安全。

我们的主角ConcurrentHashMap你应该懂了？

