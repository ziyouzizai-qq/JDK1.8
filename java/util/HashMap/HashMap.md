# HashMap源码解析

其实更多人知道哈希表，散列表，是当今世界查询速度最快的数据结构，
离不开设计，其次算法。
1.为何这样设计？
2.有哪些算法

JDK1.8的HashMap: 数组+链表+红黑树

JDK1.8之前的HashMap：数组+链表

对之前的HashMap做了优化，查询速度上的优化，当在某一程度上链表的查询速度是比红黑树快的，反之，红黑树要快，这样的设计，涉及到离散中泊松分布。下面是形成链表长度的概率

     * 0:    0.60653066
     * 1:    0.30326533
     * 2:    0.07581633
     * 3:    0.01263606
     * 4:    0.00157952
     * 5:    0.00015795
     * 6:    0.00001316
     * 7:    0.00000094
     * 8:    0.00000006



**首先我先说明一下，这个是jdk1.8的HashMap，前半段我们主要讲的是数组+链表，如果遇到红黑树的操作仅仅是知道有这个操作，只会讲为什么会有这个操作，不会细讲红黑树的方法，当把整个数组+链表捋顺之后，我们就再看看什么是红黑树，之后再研究红黑树，主要是交叉起来玩怕你们扛不住。**

#### HashMap的数据结构
```java
// HashMap存放键值对的容器，即所谓的数组
transient Node<K,V>[] table;
// 负载因子，默认值为常量0.75f
final float loadFactor;
// 倘若给一个比0.75小的数，意味着整个数组的可用空间减少，造成空间浪费
// 倘若给一个比0.75大的数，虽然空间利用率高了，但是达到更高的数量级才能触发扩容，容易产生哈希碰撞。

// 能存放元素的一个阈值，一般情况下基本上是数组长度（2的n次方）*0.75
int threshold；

// 0.75默认值
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 右移30位
static final int MAXIMUM_CAPACITY = 1 << 30;
```

#### Node的数据结构
```java
static class Node<K,V> implements Map.Entry<K,V> {
        // hash值 hash只和key肯定是固定的，final
        final int hash;
        // key 
        final K key;
        // 存放的值
        V value;
        // 下一个节点，即子节点
        Node<K,V> next;
        // 以上这个几个属性知道了就行了，下面方法简单，有些简单一眼看得出来的东西我不会讲

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

#### HashMap的构造方法
1. 无参
    ```java
    public HashMap() {
        // 负载因子 = 0.75f
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
    ```
2. 有参
    1.指定容量
    ```java
    // initialCapacity = 9
    public HashMap(int initialCapacity) {
        // DEFAULT_LOAD_FACTOR = 0.75f
        // 设计的比较有拓展性，可以自己指定负载因子
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    ```
    2.指定容量+指定负载因子
    ```java
    public HashMap(int initialCapacity, float loadFactor) {
        // initialCapacity = 9
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        //static final int MAXIMUM_CAPACITY = 1 << 30;
        // MAXIMUM_CAPACITY = 01000000 00000000 00000000 00000000 = 2的30次方
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        // 负载因子 = 0.75f
        this.loadFactor = loadFactor;
        // this.threshold = n+1 = 16
        // ★⑴
        this.threshold = tableSizeFor(initialCapacity);
    }

    /**
    * 由于数组以2的n次方设计，会将一个你指定容量(cap)大于或等于2的n次方的值返回
    * 如果指定2的n次方，遵循该规则，是不是可以直接返回。例如：指定16
    * ，就必须返回16。
    */
    static final int tableSizeFor(int cap) {
        // cap = 9
        // ★⑵
        // 有没有传过来的指定容量是16（2的n次方）
        // 假设cap不减1
        int n = cap - 1;
        // n += n+1; ===> n = n+ (n+1);
        n |= n >>> 1; // n = n | n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    ```
   
    3.将旧HashMap的元素放入新HashMap中
    ```java
    public HashMap(Map<? extends K, ? extends V> m) {
        // 负载因子 = 0.75f
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }

    /**
    *  由于putMapEntries方法多处调用，有一些情况之后讨论
    */
    // ★⑶
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        // 旧HashMap中的键值对数量
        int s = m.size();
        if (s > 0) {
            // 说明table还未初始化
            if (table == null) { // pre-size
                // 旧HashMap的table长度为16，因为负载因子，阈值是12，但是此时旧的HashMap元素个数是五个
                // ft = 7.6

                // 旧HashMap中假设元素的个数在[7, 12], 如果个数是在12,
                // 新数组长度为32，
                // 如果不加 1.0F，新数组长度也是16，元素为12，下一次再put有很大可能性resize(扩容)，所以在resize之前先减容量扩大一遍
                float ft = ((float)s / loadFactor) + 1.0F;
                // t = 7
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold) // 比阈值大就重新计算阈值
                    threshold = tableSizeFor(t);
                    // threshold = 8
            }
            else if (s > threshold) // table != null 并且元素个数大于阈值就扩容（如果阈值是12，我要放的元素小于等于12，不在这里扩容，看下面操作）
                resize();
            // 遍历旧HashMap的键值对，对于entrySet之后讲
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                // 获取key
                K key = e.getKey();
                // 获取值
                V value = e.getValue();
                // 将旧数组的值计算索引放如新数组
                // putVal放入put中讲解，此时evict为false，table创建模式
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
    ```

<font color=red>★⑴</font>在HashMap的数据结构中我曾经说过，一般情况下threshold为数租长度的0.75，但是此时的table还是null，所以呢threshold经过tableSizeFor方法后计算出的是table的长度，这个情况在第一次put值的时候，进入resize操作的时候会有对应的处理。
<font color=red>★⑵</font>至于为什么一上来先减1?详情看tableSizeFor.md文档。
<font color=red>★⑶</font>putMapEntries方法在HashMap中是由两处地方调用，一个是构造器中调用，这种情况是table == null的情况，还有就是putAll的方法，table == null和table != null情况都有可能。取决于你是否put过值，put值会将table初始化。当然，如果table == null，对于新数组的容量是可以调节的，例如：旧数组容量是64，但是只放了8个元素，在计算新数组的容量时会计算为16，原因是float ft = ((float)s / loadFactor) + 1.0F;然后与threshold（其实table=null的情况，threshold扮演的角色是table.length）比较，是否需要调节。

### HashMap的核心方法

#### put(K key, V value)
```java
public V put(K key, V value) {
    // 核心方法putVal，先看hash方法
    return putVal(hash(key), key, value, false, true);
}

// 对于HashMap来讲是允许放以null为key的值的
static final int hash(Object key) {
    int h;
    // 如果key为null，null的hash值为0
    // ★⑴
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

/**
* Implements Map.put and related methods
*
* @param hash key计算的hash值
* @param key key
* @param value value
* @param onlyIfAbsent flag，如果为true，不能覆盖已存在的值
* @param evict flag，如果为false，table为创建模式
* @return previous value, or null if none
*/
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果table为null或者table长度为0
    if ((tab = table) == null || (n = tab.length) == 0)
        // 进行扩容并且获取table的长度
        n = (tab = resize()).length;
    // ★⑵
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 如果是null，直接进行存放，没有子节点，所以next为null
        tab[i] = newNode(hash, key, value, null);
    else {
        // p不为null的情况，有可能形成链表或者红黑树
        // e节点为查询出来的节点
        Node<K,V> e; K k;
        // ★⑶
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 发现头节点就是要更新的节点，注意，set方法有可能是新建，有可能是更新
            e = p;
        else if (p instanceof TreeNode)
            // 判断是红黑树，红黑树操作
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 链表结构，开启计数，
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 作者注释第一个节点是-1啥意思
                    // p是头节点，binCount=0时，p为第2个节点，不就是-1为第一个节点[-1,6]是八个节点，一旦为7，就得尝试转红黑树
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
            // 空方法
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    // 空方法
    // 这里是留了空方法，这是很常见的设计模式，模板方法
    afterNodeInsertion(evict);
    return null;
}

// 扩容这个方法有可能太长而不愿意看，但是确实非常简单
final Node<K,V>[] resize() {
    // 获取旧的数组
    Node<K,V>[] oldTab = table;
    // 记录旧的数组长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 记录旧的阈值
    int oldThr = threshold;
    // 初始化新的数组长度和新的阈值
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
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
<font color=red>★⑴</font>(h = key.hashCode()) ^ (h >>> 16)这个就是将key对应的hashCode高16位和低16位进行异或求出一个hash值，这样充分利用了hashcode所有位，进行hash计算，求出来的hash值会更加的散列，减少hash碰撞，但是也有可能不同的key所计算的hash值是一样的，我举两个极端的例子，例1：如果两个key的hashcode值一样，所对应hash值一定一样。例2：如果两个key高16位和低16位互为相反，意味着hashcode不一样，但是计算的hash值是一样的。这些都会产生hash冲突，但是这种情况还是很罕见的。
<font color=red>★⑵</font>(p = tab[i = (n - 1) & hash]) == null，这行代码，心坎(n - 1) & hash，这里的n是tab.length，有可能是resize之后的n，也有可能是if判断中获取的n，总之，高手写的代码就是这么的灵活妙哉。多学学这样的代码风格。之前在讨论table.length时，我一直说table的长度一定是2的n次方，此时n-1，对于数组来讲，这是数组下标的最大值，下标选择是[0,table.lenth-1]。对于二进制来讲，后位全是1，进行与的时候，所计算出的下标的决定权不在table身上，决定权在于key计算的hash值和n-1所对应的位01来决定，(n - 1) & hash这时应该与运算，相当于十进制中的取模，位运算计算的效率肯定远高于十进制，所以用在HashMap中，肯定是选用前者，毕竟map肯定是追求整体的一个速率的。计算出的索引值查看所对应的p节点是否为null。
<font color=red>★⑶</font>p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k)))，这个判断很常见，如果key产生了hash碰撞，还需要判断是否是同一个引用或者是equals是否相等，这就是为什么HashMap所用到的key一定要成对重写hashcode和equals方法。这里有一个细节，由于HashMap是允许以null为key的，所以需要判断，但是下面for中也对key作了null判断，其实这里我们可以发现，null为key是不会产生链表的，而且如果key为null，只能被第一个if条件捕捉到，下面看上去是没有必要再判断防止出现空指针，因为null为key根本不可能走到下面，但是作者还是写了，显然是为了代码健壮性。

