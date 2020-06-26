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

<font color=red>★⑴</font>在HashMap的数据结构中我曾经说过，一般情况下threshold为数租长度的0.75，但是此时的table还是null，所以呢threshold经过tableSizeFor方法后计算出的是table的长度，这个情况在第一次put值的时候，进入resize操作的时候会有对应的处理。<br>
<font color=red>★⑵</font>至于为什么一上来先减1?详情看tableSizeFor.md文档。<br>
<font color=red>★⑶</font>putMapEntries方法在HashMap中是由两处地方调用，一个是构造器中调用，这种情况是table == null的情况，还有就是putAll的方法，table == null和table != null情况都有可能。取决于你是否put过值，put值会将table初始化。当然，如果table == null，对于新数组的容量是可以调节的，例如：旧数组容量是64，但是只放了8个元素，在计算新数组的容量时会计算为16，原因是float ft = ((float)s / loadFactor) + 1.0F;然后与threshold（其实table=null的情况，threshold扮演的角色是table.length）比较，是否需要调节。<br>

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
    // 如果旧的数组长度存在
    if (oldCap > 0) {
        // 如果超过最大长度
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 阈值为int最大，允许放满
            threshold = Integer.MAX_VALUE;
            // 无法扩容了，返回旧数组
            return oldTab;
        }
        // oldCap扩大2倍还小于MAXIMUM_CAPACITY并且oldCap大于等16
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 阈值扩大2倍
            newThr = oldThr << 1; // double threshold
    }
    // oldCap = 0，并且此时oldThr>0
    // ★⑷
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
    // oldCap = 0，并且此时oldThr = 0
    // ★⑸
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 处理剩余情况的newThr
    if (newThr == 0) {
        // 对★⑷情况处理，还有对oldCap < 16情况处理
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    // 新的阈值赋上去
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 定义新长度的新数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    // 新数组
    table = newTab;
    // 如果旧的数组不为null，说明不是第一次扩容
    if (oldTab != null) {
        // 遍历旧数组术后存在元素
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // 节点不为null，说明要重新在新数组中计算存放
            if ((e = oldTab[j]) != null) {
                // 设置旧的元素为null，防止内存泄露，e引用着，此对象不会被回收
                oldTab[j] = null;
                // 如果没有子节点
                if (e.next == null)
                    // 直接在新数组中计算自己的下标并且赋值上去
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode) // 红黑树操作
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 链表操作
                    // 链条lo 头              尾
                    Node<K,V> loHead = null, loTail = null;
                    // 链条hi 头              尾
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // 这里计算索引原理详见resize.md
                    // ★⑹
                    do {
                        // 获取子节点
                        next = e.next;
                        // 第一次进来e还是头节点，请注意
                        if ((e.hash & oldCap) == 0) {
                            // 尾还是null，说明无元素
                            if (loTail == null)
                                // 头元素
                                loHead = e;
                            else
                                // 子元素
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
                    // 尾元素为null，说明无元素
                    if (loTail != null) {
                        // 无子节点
                        loTail.next = null;
                        // 放原数组位置
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 放原数组索引位置+旧的数组长度
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    // 返回数组
    return newTab;
}
```
<font color=red>★⑴</font>(h = key.hashCode()) ^ (h >>> 16)这个就是将key对应的hashCode高16位和低16位进行异或求出一个hash值，这样充分利用了hashcode所有位，进行hash计算，求出来的hash值会更加的散列，减少hash碰撞，但是也有可能不同的key所计算的hash值是一样的，我举两个极端的例子，例1：如果两个key的hashcode值一样，所对应hash值一定一样。例2：如果两个key高16位和低16位互为相反，意味着hashcode不一样，但是计算的hash值是一样的。这些都会产生hash冲突，但是这种情况还是很罕见的。<br>
<font color=red>★⑵</font>(p = tab[i = (n - 1) & hash]) == null，这行代码，心坎(n - 1) & hash，这里的n是tab.length，有可能是resize之后的n，也有可能是if判断中获取的n，总之，高手写的代码就是这么的灵活妙哉。多学学这样的代码风格。之前在讨论table.length时，我一直说table的长度一定是2的n次方，此时n-1，对于数组来讲，这是数组下标的最大值，下标选择是[0,table.lenth-1]。对于二进制来讲，后位全是1，进行与的时候，所计算出的下标的决定权不在table身上，决定权在于key计算的hash值和n-1所对应的位01来决定，(n - 1) & hash这时应该与运算，相当于十进制中的取模，位运算计算的效率肯定远高于十进制，所以用在HashMap中，肯定是选用前者，毕竟map肯定是追求整体的一个速率的。计算出的索引值查看所对应的p节点是否为null。<br>
<font color=red>★⑶</font>p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k)))，这个判断很常见，如果key产生了hash碰撞，还需要判断是否是同一个引用或者是equals是否相等，这就是为什么HashMap所用到的key一定要成对重写hashcode和equals方法。这里有一个细节，由于HashMap是允许以null为key的，所以需要判断，但是下面for中也对key作了null判断，其实这里我们可以发现，null为key是不会产生链表的，而且如果key为null，只能被第一个if条件捕捉到，下面看上去是没有必要再判断防止出现空指针，因为null为key根本不可能走到下面，但是作者还是写了，显然是为了代码健壮性。<br>
<font color=red>★⑷</font>之前我说过，指定容量构造的时候，第一次扩容table为null，此时的threshold为数组长度的，所以旧的阈值直接赋给了新容量。<br>
<font color=red>★⑸</font>这种情况就是大家最常用的无参构造器第一次扩容的情况，默认值是16长度，阈值12由此而来。<br>
<font color=red>★⑹</font>这里一定看resize文档，看下来绝对会恍然大悟，而且非常简单，所以下面的源码我只用少量的笔墨去分析，分析一半就可以了。

**如果上面的源码分析都可以理解透彻的话，心里应该就只剩下一个疑问，如何去转红黑树，如何操控红黑树，没有基础的甚至会问红黑树是什么？**

### 转红黑树

**转红黑树是在putVal方法中对链表操作中，当迭代了8个节点发现不是更新操作，会生成第9个元素，此时有可能转红黑树，在这里我仅仅说有可能，对准备转红黑树的方法treeifyBin中，我们可以找到答案。**

#### treeifyBin(Node<K,V>[] tab, int hash)

```java
// tab：当前数组 hash：key的hash值
final void treeifyBin(Node<K,V>[] tab, int hash) {
    // n：记录数组的长度
    int n, index; Node<K,V> e;
    // 尽管tab在进入这个方法前大多数情况不为null，因为HashaMap线程不安全问题，代码的健壮性还是要有的
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        // 之前我说过，超过8个元素不一定转红黑树，有可能会扩容，前提是数组的长度小于64(MIN_TREEIFY_CAPACITY)
        // ★⑴
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) { 
        // 这里由于索引值一样，官方可太“坏”了，拿尾节点的hash值来获取头节点
        // hd指向头节点，t1记录前一个节点，与后节点需要互相依赖
        TreeNode<K,V> hd = null, tl = null;
        do {
            // 将节点放入包装出一个TreeNode对象
            TreeNode<K,V> p = replacementTreeNode(e, null);
            // 如果t1是null，说明是第一次
            if (tl == null)
                // 头节点赋值
                hd = p;
            else {
                // 当前树节点前节点引用指向用t1记录的前节点
                p.prev = tl;
                // 前节点的后引用指向当前节点，形成一个互相依赖，双向链表的关系
                tl.next = p;
            }
            // 此时用t1记录前一个节点,为后一个节点时互相使用
            tl = p;
            // 迭代节点
        } while ((e = e.next) != null);
        // ★⑵
        // 将该位置的链表变成ThreeNode的双向链表放入，HashMap毕竟线程不安全，代码的健壮性还是要有的
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}

// For treeifyBin
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    // 说到底这边只是包了一层TreeNode，底层还是用的Node来保存数据，所以还是Node类型
    return new TreeNode<>(p.hash, p.key, p.value, next);
}

// TreeNode继承了LinkedHashMap.Entry
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V>{

    // 父节点
    TreeNode<K,V> parent;
    // 左右节点
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    // 
    TreeNode<K,V> prev;
    // 颜色flag
    boolean red;
    // Node父类来保存hash, key, val, next
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }

    // 转红黑树操作，传入tab
    final void treeify(Node<K,V>[] tab) {
        // 根节点
        TreeNode<K,V> root = null;
        // hd调的，肯定从this开始
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            //获取子节点
            next = (TreeNode<K,V>)x.next;
            // 保证当前节点左右节点为空
            x.left = x.right = null;
            if (root == null) {
                // 头节点无父节点
                x.parent = null;
                // 颜色是黑色
                x.red = false;
                // 根节点赋值
                root = x;
            }
            else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = root;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                                (kc = comparableClassFor(k)) == null) ||
                                (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);

                    TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        root = balanceInsertion(root, x);
                        break;
                    }
                }
            }
        }
        moveRootToFront(tab, root);
    }
}

// LinkedHashMap.Entry有继承了HashMap.Node，所以TreeNode是Node类型
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

```

<font color=red>★⑴</font>首先在链表和红黑树之间的选择，无非都是查询效率的问题。如果产生链表，就必须从头节点一层层next，时间复杂度是O(n)，这种查询效率非常糟糕，所以减少hash碰撞就是为了提高HashMap的性能。但是又不可避免会出现hash碰撞，形成链表。所以官方通过大量的实验测试来制定了HashMap重要参数，例如负载因子0.75，数组长度以2的次方幂等。而且采取了链表长度大于8并且数组长度大于等于64，转红黑树的效率会提升等优化措施，并且建议HashMap的所存放的数据量是可以提前预估好的，通过指定容量，就不会频繁扩容（扩容机制）影响性能。在这里链表长度大于8并且数组长度小于64也是一种优化措施，HashMap会认为长度在64以下的没有必要去转红黑树，先将其扩容，之前对于我在扩容分析如果吃的很透的话，我下面举的例子会轻松理解，首先我补一个细节，当进入treeifyBin方法之前，第9个元素已经放在第8的元素的next中了，p.next = newNode(hash, key, value, null);这是源码。例：假设数组的长度为16，在很极端的情况，我给出的8个hash值不同的元素所计算的索引值一样(只需要各个元素hash值后4位一样就行)，形成一个长度为8的链表，如果此时我在放一个元素，就很极端，索引值和这8个元素一样，成为链表的第9个节点，调用treeifyBin后发现，长度不满足64，只能resize，之前我说过，数组扩容两倍后，有可能将链表散到两个位置，一个是原数组位置，一个是原数组索引+原数组长度。有可能长度为9的链表，就会形成，例如像4,5长度的两个链表。这样查询效率也会提升吧。这个时候我在问大家一个问题：HashMap有没有可能形成长度超过8的链表，接着上面的例子，不一定是长度是4,5链条链表，也有可能是长度为9的一条链表，这说明这9个元素的hash值后5位是一样的，所以说以长度超过8的链表也是有可能的。<br />

<font color=red>★⑵</font>所以这里还有一个注意点，链表如何去转红黑树的，这段通过尾节点索引获取头节点，进行迭代转换ThreeNode类型，并转成双向链表的转红黑树过渡过程。大家也不要轻易去忽略，看源码和背知识点或者是看人家阉割版培训机构源码解析视频有一个很大的区别就是，你可以将源码中作者使用的很多细节在你的回答下体现出来。而不是一上来就说这个过程是怎么做的原理是什么，将过渡过程省略，导致这里的知识空缺，这还是很危险的，假设是我面试你呢？是自己分析的源码和背知识点的人是一目了然的，当然，除非有些面试官自己也不是太懂，这个除外。在补充一点，像很多容器类啊，什么链表啊，很常见，如果是感觉很吃力的话，分为两种人，第一种人是各个引用和对象的关系不能分的特别清楚，第二种就是数据结构和算法知识比较贫乏，这里我没有特别好的建议，一个就是看相关书，一个就是刷算法，刷题可以自己去leetcode上去练习数据结构方面特别是链表这块的题目。上面我也提过培训机构阉割版源码解析视频只能是辅助你，启发你，你并不能说看完就懂，更何况大多数培训机构为了都是赚钱，挂羊头卖狗肉又不是一次两次了，关键是这狗肉也不给你剁碎了，让你可以自己嚼。