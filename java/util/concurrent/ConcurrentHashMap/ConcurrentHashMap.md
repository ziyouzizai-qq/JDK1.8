# ConcurrentHashMap

## 1.简介

​	HashMap虽然是查询速度最快的K-V容器，但是却又是一个线程不安全的容器。因此JDK虽然有HashTable这样早期线程安全的容器（其原理仅仅是通过synchronized去做线程同步），由于HashTable是锁整个map结构，其细粒度很低，因此并发效率极低，并不能满足超高并发和高并发场景。而在JDK5 Doug Lea的juc出来之后，ConcurrentHashMap就将HashTable给取代了，一开始是通过分段锁去提高并发效率，之后采取CAS自旋+synchronized进一步提高细粒度，并发效率极高。所以ConcurrentHashMap在多线程环境中很常见，知道其中原理非常重要。

## 2.源码解析

ConcurrentHashMap数据结构还是和HashMap一样

### 2-1.构造器

```java
// 无参构造
public ConcurrentHashMap() {
}

// 指定容量构造
public ConcurrentHashMap(int initialCapacity) {
    // 默认0.75的负载因子和1的并发等级
    this(initialCapacity, LOAD_FACTOR, 1);
}

// 参数1：初始化容量
// 参数2：负载因子
// 参数3：并发等级
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    // 检查参数是否异常
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    // 如果并发等级大于初始化容量，并发等级就是初始化容量
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins(至少使用同样的箱子)
        initialCapacity = concurrencyLevel;   // as estimated threads
    // 获取table的长度，指定初始化容量除以负载因子
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    // 获取table实际容量，tableSizeFor根据size推算出比size大的最小2的n次方
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    // 设置实际容量
    // sizeCtl为当前table允许放的元素个数，此时是初始化table的实际长度，和HashMap中threshold一样
    this.sizeCtl = cap;
}
```

### 2-2.put方法

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

// onlyIfAbsent的作用：如果当前位置已存在一个值，是否替换，false是替换，true是不替换
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ConcurrentHashMa中不允许存放k-value为null的情况
    if (key == null || value == null) throw new NullPointerException();
    // 获取key的hashcode，用于计算hash值((h ^ (h >>> 16)) & HASH_BITS)
    int hash = spread(key.hashCode());
    int binCount = 0;
    // 获取table，自旋去执行put操作，因为有可能会put失败
    // 例如初始化数组时由于别的线程初始化完后，就需要重新获取table了
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh; K fk; V fv;
        if (tab == null || (n = tab.length) == 0)
            // 如果table不存在或者长度为0，初始化table
            // 有可能多个线程，当前线程由于table为null进入initTable，还没进，其他线程就初始化完毕了
            // 则当前线程在initTable中不处理，仅仅把最新的table拿出来
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 数组存在，且计算的索引值的位置没有值
            // CAS在这个位置存值，由于hash冲突，有可能不同的key在同一个索引位置放值，因此存在线程安全问题
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                // 如果CAS成功，put就成功，退出循环
                break;                   // no lock when adding to empty bin
                // 上面这个英文注释意思是，将Map每一个索引值的位置当做一个bin，当这个bin为空时，
                // 是不加lock的，CAS不是lock，因为在空的bin中只有一个线程会CAS成功，其他失败了重新获取table
                // 重新计算位置，如果在不扩容的情况下，应该还是该位置，但是此时该bin不为空了，则会产生链表
                // else 分支存在链表操作
        }
        else if ((fh = f.hash) == MOVED) // MOVED = -1
            // 如果f头的节点是ForwardingNode，这是一个内部类，后面再讲
            tab = helpTransfer(tab, f);
        else if (onlyIfAbsent // check first node without acquiring lock
                 && fh == hash // hash值一样
                 && ((fk = f.key) == key || (fk != null && key.equals(fk))) // 并且key是同一个或者equal相同
                 && (fv = f.val) != null) // 并且存放的值不为null,我在想值还有null的情况，put时候不应该空指针？
            // 这些疑问我后面做一个归纳
            // onlyIfAbsent为true，可以看见只是将map中对应key的value返回，没有做更新操作
            // onlyIfAbsent为false根本进不来
            return fv;
        else {
            // 剩余的情况
            // 新增节点返回null
            V oldVal = null;
            // 将对这个节点加锁，这就是和Hashtable的区别，这是锁bin，后者锁整个map
            synchronized (f) {                                                                                               // 再次获取该位置的节点，如果和f一致，说明该位置的头节点没有变化
                // 这是防止当前线程在获取到索引位置的节点后，由于线程上下文切换，其他线程在此期间改变map结构
                // 或者是remove了该节点，因此在对该节点加锁之后还需要确认一下table中该索引位置的节点和之前取到的节点
                // 引用一致，方可继续执行，如果不一致，则认为put失败，就需要重新获取table进行put
                if (tabAt(tab, i) == f) {
                    // 如果hash值大于0，说明是普通的Node
                    // 如果是非普通的
                    // TreeBin类型固定为-2
                    // ReservationNode类型固定是-3，和上面ForwardingNode同理
                    if (fh >= 0) {
                        // 由于头节点算一个，所以binCount开始为1
                        binCount = 1;
                        // 一边迭代寻找，一边++binCount
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 获取值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                // 获取旧值
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    // 更新替换
                                    e.val = value;
                                // 更新完毕，退出循环
                                break;
                            }
                            Node<K,V> pred = e;
                            // 进行迭代，如果发现迭代到最后，不是更新，是新增
                            if ((e = e.next) == null) {
                                // 则在最后加一个节点
                                pred.next = new Node<K,V>(hash, key, value);
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
                    else if (f instanceof ReservationNode)
                        // 如果是保留节点，抛出异常
                        throw new IllegalStateException("Recursive update");
                }
            }
            // binCount如果不为0，说明map结构发生改变，准确的说对应bin发生改变
            if (binCount != 0) {
                // 如果大于8转红黑树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                // 旧值不为null，直接返回
                if (oldVal != null)
                    return oldVal;
                // 跳出循环
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}

// 计算hash值
static final int spread(int h) {
    // 高16位与低16位异或后的值与int的最大值与
    return (h ^ (h >>> 16)) & HASH_BITS;
}

// 初始化table，开始产生线程安全问题
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // table为空或者长度为0，自旋，否则，不做处理
    while ((tab = table) == null || tab.length == 0) {
        // 获取sizeCtl的值，如果发现sizeCtl为-1，即小于0，则让出CPU执行
        // 因为此时有其他线程CAS成功，则当前线程初始化失败，仅仅做一次空旋转
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) { // CAS
            try {
                // 为什么CAS成功后还需要if判断。由于线程上下文切换。会出现这种情况
                // 当进入while循环还是table为null，但是进入之后，当前线程由于上下文切换
                // 停在if ((sc = sizeCtl) < 0)的代码块，这块代码块还没执行，此刻别的线程sizeCtl
                // 获取到CPU执行时间片，CAS成功，设置-1，相当于加了一把独占锁，并且初始化完毕table后，
                // 再重新设置sizeCtl，finally中sizeCtl = sc，sizeCtl是大于0的
                // 时间片轮到当前线程执行时，从if ((sc = sizeCtl) < 0)开始执行，sizeCtl>0,因此不会初始化失败
                // 因此在后面CAS也能成功，但是如果不加判断的话，就会将第一个线程存在值的table替换掉，当然，存放的
                // 键值对也会丢失掉。
                // 这个初始化失败仅仅是当一个线程通过CAS将值设置成-1，然后又改为正常值，这个期间相当于一把独占锁。
                // 因此不在该期间时，是无法得知table是否被其他线程初始化，因此在当前线程CAS成功后，还需判断。如果table
                // 存在，存放值的操作就不在这边了，只有初始化table的那个线程才有权利在这个方法中存值。
                if ((tab = table) == null || tab.length == 0) {
                    // 进行初始化table，sc<0上面会Thread.yield();
                    // 因此sc至此只会>=0,所以当sizeCtl为0时，说明没有通过构造器，默认容量是16
                    // 如果通过构造器设置过，容器为sizeCtl
                    // 因此sizeCtl既是table的实际长度，也是线程同步的值
                    // 但是sizeCtl其实是0.75的阈值，因为在初始化tables时候，此时是table的长度，为2的n次方
                    // 一旦初始化完成后，值就会变为原来的0.75.因此sc = n - (n >>> 2)，然后finally中重新设置sizeCtl
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    // new 一个node数组，并且将n放入，可以看见数组长度为n
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    // table设置
                    table = tab = nt;
                    // sc的值为原值的0.75，我不知道为什么这边为什么写死了，而不是通过负载因子。如果负载因子不是0.75呢
                    sc = n - (n >>> 2);
                }
            } finally {
                // 重新设置，-1值独占锁，不需要CAS，volatile保证可以对其他线程立即可见
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

