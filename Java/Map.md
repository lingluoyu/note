## HashMap

### HashMap数据结构

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

	// 默认初始化容量，16，必须为2的幂
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    // 最大容量，2^30
    static final int MAXIMUM_CAPACITY = 1 << 30;
    // 负载因子，默认为0.75
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // 链表转为树的阈值
    static final int TREEIFY_THRESHOLD = 8;
    // 树转为链表的阈值
    static final int UNTREEIFY_THRESHOLD = 6;
    // 只有当 HashMap 中的数组大小达到此阈值，才会将冲突的元素由链表转为树；若没有达到则首先扩容数组
    // MIN_TREEIFY_CAPACITY的值至少是TREEIFY_THRESHOLD的4倍。
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    static class Node<K,V> implements Map.Entry<K,V> {
        ...
    }
    
    // 存储数据的数组
    transient Node<K,V>[] table;
    // 遍历的容器
    transient Set<Map.Entry<K,V>> entrySet;
    // Map中KEY-VALUE的数量
    transient int size;
    /**
     * 结构性变更的次数。
     * 结构性变更是指map的元素数量的变化，比如rehash操作。
     * 用于HashMap快速失败操作，比如在遍历时发生了结构性变更，就会抛出ConcurrentModificationException。
     */
    transient int modCount;
    // 下次resize的操作的size值。
    int threshold;
    // 负载因子，resize后容量的大小会增加现有size * loadFactor
    final float loadFactor;
}
```

### HashMap初始化

```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // 其他属性均为默认值
}
```

此处并未进行数组 table 的初始化，而是在 `put()` 方法中进行初始化，防止初始化 HashMap 之后不使用浪费内存。

### HashMap存储元素

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

#### hash(key)

对key进行hash计算，确定数组索引位置

```java
static final int hash(Object key) {
    int h;
    // h = key.hashCode() 通过 java 本地方法取hashcode
    // h ^ (h >>> 16) 将 hashcode 高16位参与运算
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

通过 hashcode 的高16位异或低16位得到最终的 hash 值：(h = key.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。

得到 hash 值，数值为 int 类型，取值范围为‑2147483648到2147483648，需要对 hash 值进行取模运算来确定数组下标。伪代码如下：

```java
bucketIndex = indexFor(hash, table.length);
// 把散列值和数组长度做"与"运算
static int indexFor(int h, int length) {
   return h & (length-1);
}
```

**为什么HashMap的数组长度要取2的整次幂？**

因为这样（数组长度‑1）正好相当于一个“低位掩码”。“与”操作的结果就是散列值的高位全部归零，只保留低位值，用来做数组下标访问。以初始长度16为例，16‑1=15。2进制表示是000000000000000000001111。和某散列值做“与”操作如下，结果就是截取了最低的四位值。

```bash
  10100101 11000100 00100101
& 00000000 00000000 00001111
----------------------------------
  00000000 00000000 00000101 //高位全部归零，只保留末四位
```

**扰动函数**

若只取最后几位散列值，会产生严重的碰撞，因此通过 `(h = key.hashCode()) ^ (h >>> 16)` 对 hashCode 进行扰动，降低碰撞的可能性，同时保留了原 hashCode 的高位和低位信息。

#### putVal

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table 为空或初始化为0，使用默认值初始化 table
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 对应数组下标没有元素，生成头结点并放入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 对应数组下标有元素，数据冲突，需要解决冲突
    else {
        Node<K,V> e; K k;
        // put 元素和已存在元素是否相同（hash 相同且 equals 返回 true）
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // put的元素和已经存在的元素是不相同
        // 若当前数组下标中所存元素为树类型，则将元素放入树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 当前数组下标中所存元素为链表，则将元素放入链表最后一个元素
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 当前数组下表元素超过转换为树的阈值，将存储元素的数据结构变为树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 链表中已存在元素，停止遍历
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 已存在元素，替换value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 若元素数量大于阈值，进行扩容 resize 操作
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

#### treeifyBin

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
	// table 长度不超过64，首先进行 table 扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    // table 长度 > MIN_TREEIFY_CAPACITY=64，链表转为树
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

#### resize

HashMap的扩容机制用的很巧妙，以最小的性能来完成扩容。扩容后的容量就变成了变成了之前容量的2倍，初始容量为16，所以经过rehash之后，元素的位置要么是在原位置，要么是在原位置再向高下标移动上次容量次数的位置，也就是说如果上次容量是16，下次扩容后容量变成了16+16，如果一个元素在下标为7的位置，下次扩容时，要不还在7的位置，要不在7+16的位置。

**Why?**

元素在存入原 table 中的时候，使用 `(n - 1) & hash` 确定存入 table 的下标，table 的大小为2的整数次幂，扩容为2倍，所以 n-1 的mask范围在高位多1bit，因此新的 index 就为 **原位置 + oldCap**。因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”。而hash值的高位是否为1，只需要和扩容后的长度做与操作就可以了，因为扩容后的长度为2的次幂，所以高位必为1，低位必为0，如10000这种形式，源码中实现为`e.hash & oldCap`。

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 计算新的容量值和下一次要扩展的容量
    if (oldCap > 0) {
        // table 超过最大阈值，不进行扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // table 为超过最大阈值，扩容为2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 计算下一次 resize 的元素上限阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 将原 table 中元素移动到新的 table 中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 只有一个元素，计算 hash 值并放入新 table
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 树结构，若树中元素 < UNTREEIFY_THRESHOLD=6，则会转换为链表
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 链表数据机结构
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 利用哈希值 与 旧的容量，可以得到哈希值去模后，是大于等于oldCap还是小于oldCap，等于0代表小于oldCap，应该存放在低位，否则存放在高位
                        // hash碰撞后高位为0，放入低Hash值的链表中
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // hash碰撞后高位为1，放入高Hash值的链表中
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 低hash值的链表放入数组的原始位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 高hash值的链表放入数组的原始位置 + 原始容量
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

## ConcurrentHashMap

### ConcurrentHashMap数据结构

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    // 最大元素容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    // 默认初始容量
    private static final int DEFAULT_CAPACITY = 16;
    // 最大数组长度
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    // 默认并发数量
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    // 默认负载因子
    private static final float LOAD_FACTOR = 0.75f;
    //转换为红黑树的阈值
    static final int TREEIFY_THRESHOLD = 8;
    //退化成链表的阈值
    static final int UNTREEIFY_THRESHOLD = 6;
    // 冲突元素由链表转为树的最小 table 大小
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    // 保存key-value的数据结构
    static class Node<K,V> implements Map.Entry<K,V> {
        ...
    }
    
    // 数组
    transient volatile Node<K,V>[] table;
    // 扩容临时 table
    private transient volatile Node<K,V>[] nextTable;
    // 控制标识符，用来控制table初始化和扩容操作的，在不同的地方有不同的用途，其值也不同，所代表的含义也不同
    // 负数代表正在进行初始化或扩容操作
    // -1代表正在初始化
    // -N 表示有N-1个线程正在进行扩容操作
    // 正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小
    private transient volatile int sizeCtl;
    
    // 一个特殊的Node节点，hash值为-1，其中存储nextTable的引用。只有table发生扩容的时候，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为null或则已经被移动
    static final class ForwardingNode<K,V> extends Node<K,V> {
        ...
    }
}
```

### ConcurrentHashMap构造函数

ConcurrentHashMap 构造函数进行基础属性设置，不进行初始化。

```java
public ConcurrentHashMap() {
}

public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

### ConcurrentHashMap方法

ConcurrentHashMap初始化发生在插入元素时，调用 `initTable()` 方法进行初始化。

#### putVal

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key 和 value 均不能为空
    if (key == null || value == null) throw new NullPointerException();
    // 获取 hash
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // table 为空，初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // tab[i] 没有元素，直接插入，无需加锁
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 有线程正在进行扩容，则先帮助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 对该节点进行加锁处理（hash值相同的链表的头节点）
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // fh > 0 表示为链表，将该节点插入到链表尾部
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // hash 和 key 都一样，替换value
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 链表尾部  直接插入
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 树节点，按照树的插入操作进行插入
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
                // 链表长度 > TREEIFY_THRESHOLD = 8，将链表转为树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // size + 1
    addCount(1L, binCount);
    return null;
}
```

#### initTable

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl < 0 有其他线程在初始化，挂起当前线程
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // 当前线程得到初始化 table 的机会，通过 CAS 将 sizeCtl 设为-1，标识本线程正在进行初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 下次扩容大小
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

#### get

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 计算hash
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
        // 搜索到的节点key与传入的key相同且不为null,直接返回这个节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 树
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 链表，遍历
        while ((e = e.next) != null) {
            if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

#### size

```java
// ConcurrentHashMap中元素个数,但返回的不一定是当前Map的真实元素个数。基于CAS无锁更新
private transient volatile long baseCount;
// 存储每个 table 元素个数
private transient volatile CounterCell[] counterCells;

@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

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
            // 遍历，所有counter求和
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

sumCount()就是迭代counterCells来统计sum的过程。put操作时，会影响size()，CouncurrentHashMap在put()方法最后会调用addCount()方法，该方法主要做两件事，一件更新baseCount的值，第二件检测是否进行扩容。

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // s = b + x，完成baseCount++操作
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            // 多线程CAS发生失败时执行
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    // 检查是否进行扩容
    ...
}
```

#### 扩容

当ConcurrentHashMap中table元素个数达到了容量阈值（sizeCtl）时，则需要进行扩容操作。在put操作时最后一个会调用addCount(long x, int check)，该方法主要做两个工作：1.更新baseCount；2.检测是否需要扩容操作。

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // s = b + x，完成baseCount++操作
    ...
    // check >= 0 需要进行扩容
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
            // 当前线程是唯一的或是第一个发起扩容的线程  此时nextTable=null
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                // 扩容
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

transfer()方法为ConcurrentHashMap扩容操作的核心方法。由于ConcurrentHashMap支持多线程扩容，而且也没有进行加锁，所以实现会变得有点儿复杂。整个扩容操作分为两步：

1. 构建一个nextTable，其大小为原来大小的两倍，这个步骤是在单线程环境下完成的
2. 将原来table里面的内容复制到nextTable中，这个步骤是允许多线程操作的，所以性能得到提升，减少了扩容的时间消耗

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 每核处理的量小于16，则强制赋值16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            // 构建一个nextTable对象，其容量为原来容量的两倍
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
    // 连接点指针，用于标志位（fwd的hash值为-1，fwd.nextTable=nextTab）
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 当advance == true时，表明该节点已经处理过了
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 控制 --i ,遍历原hash表中的节点
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 用CAS计算得到的transferIndex
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
            // 已经完成所有节点复制了
            if (finishing) {
                nextTable = null;
                table = nextTab;        // table 指向nextTable
                sizeCtl = (n << 1) - (n >>> 1);     // sizeCtl阈值为原来的1.5倍
                return;     // 跳出死循环，
            }
            // CAS 更扩容阈值，在这里面sizectl值减一，说明新加入一个线程参与到扩容操作
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 遍历的节点为null，则放入到ForwardingNode 指针节点
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // f.hash == -1 表示遍历到了ForwardingNode节点，意味着该节点已经处理过了
        // 这里是控制并发扩容的核心
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 节点加锁
            synchronized (f) {
                // 节点复制工作
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // fh >= 0 ,表示为链表节点
                    if (fh >= 0) {
                        // 构造两个链表  一个是原链表  另一个是原链表的反序排列
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
                        // 在nextTable i 位置处插上链表
                        setTabAt(nextTab, i, ln);
                        // 在nextTable i + n 位置处插上链表
                        setTabAt(nextTab, i + n, hn);
                        // 在table i 位置处插上ForwardingNode 表示该节点已经处理过了
                        setTabAt(tab, i, fwd);
                        // advance = true 可以执行--i动作，遍历节点
                        advance = true;
                    }
                    // 如果是TreeBin，则按照红黑树进行处理，处理逻辑与上面一致
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

                        // 扩容后树节点个数若<=6，将树转链表
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

1. 为每个内核分任务，并保证其不小于16
2. 检查nextTable是否为null，如果是，则初始化nextTable，使其容量为table的两倍
3. 死循环遍历节点，知道finished：节点从table复制到nextTable中，支持并发，请思路如下：
   - 如果节点 f 为null，则插入ForwardingNode（采用Unsafe.compareAndSwapObjectf方法实现），这个是触发并发扩容的关键
   - 如果f为链表的头节点（fh >= 0）,则先构造一个反序链表，然后把他们分别放在nextTable的i和i + n位置，并将ForwardingNode 插入原节点位置，代表已经处理过了
   - 如果f为TreeBin节点，同样也是构造一个反序 ，同时需要判断是否需要进行unTreeify()操作，并把处理的结果分别插入到nextTable的i 和i+nw位置，并插入ForwardingNode 节点
4. 所有节点复制完成后，则将table指向nextTable，同时更新sizeCtl = nextTable的0.75倍，完成扩容过程

在多线程环境下，ConcurrentHashMap用两点来保证正确性：ForwardingNode和synchronized。当一个线程遍历到的节点如果是ForwardingNode，则继续往后遍历，如果不是，则将该节点加锁，防止其他线程进入，完成后设置ForwardingNode节点，以便要其他线程可以看到该节点已经处理过了，如此交叉进行，高效而又安全。

在put操作时如果发现fh.hash = -1，则表示正在进行扩容操作，则当前线程会协助进行扩容操作。

```java
else if ((fh = f.hash) == MOVED)
    tab = helpTransfer(tab, f);
```

helpTransfer()方法为协助扩容方法，当调用该方法的时候，nextTable一定已经创建了，所以该方法主要则是进行复制工作。

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
                (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

