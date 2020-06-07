# ThreadLocal源码解析

**灵魂三问**
1. ThreadLocal如何保证线程数据互不影响，彼此独立？
2. ThreadLocal为什么会内存泄露？
3. ThreadLocal内存泄露的解决方案？

**下面带着以上三个问题从ThreadLocal源码入手，或许能解决你心中的疑惑**

****

## ThreadLocal


### ThreadLocal的实例方法
1. set(T value)
```java
public void set(T value) {
    // 首先，需要通过ThreadLocal实例进入set方法，在次获取调用本方法的线程
    Thread t = Thread.currentThread();
    // 通过该线程获取该线程所对应的threadLocals，看Thread的源码得知
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    // 获取线程对应的threadLocals
    return t.threadLocals;
}
```

<hr>

## Thread

### Thread的数据结构（挑选的重要的，并不是全部）
```java
// 由此可以发现对于每一个线程都会存在一个ThreadLocalMap的容器来存放我们的数据
// 所以问题1，线程数据互不影响，彼此独立就可以解释的通了，然后我们再来看ThreadLocalMap是什么
ThreadLocal.ThreadLocalMap threadLocals = null;
```

<hr>

## ThreadLocalMap
**ThreadLocalMap是ThreadLocal的静态内部类**

### ThreadLocalMap的数据结构
```java
/**
* The initial capacity -- MUST be a power of two.
* 初始化容量
*/
private static final int INITIAL_CAPACITY = 16;

/**
* The table, resized as necessary.
* table.length MUST always be a power of two.
* Entry类型的数组，Entry是ThreadLocalMap的静态内部类，和HashMap的Entry类没有任何关系
*/
private Entry[] table;

/**
* The number of entries in the table.
* 记录元素的个数
*/
private int size = 0;

/**
* The next size value at which to resize.
* table的阈值
*/
private int threshold; // Default to 0
```

<hr>

## Entry
**Entry是ThreadLocalMap的静态内部类**

<font color=red>当我文档写到这里的时候，我发现我不能回过头去对ThreadLocalMap做大量的源码分析，由于ThreadLocalMap会有些很让人不能理解的操作，这主要是因为Entry。所以Entry的源码分析的位置不对，在整个文档的最后面，但如果将Entry源码分析的位置放在第一个，我又感觉很突兀，所以我会根据我能让读者看得懂源码，坚持的下去的思路去解析源码。所以整个markdown文档的阅读顺序是自上而下的。</font>

**首先从Entry的类结构我们来对问题2和问题3做一个简单的回答**
1. <font color=red>Q：什么是弱引用？</font><br>
   A：java里面的引用分为强软弱虚四个引用强度，强引用，我们平时new的对象，没有引用指向才能被回收；软引用，当整个堆内存不够时，java虚拟机会尝试回收掉软引用的对象，如果回收了内存还不够，报OOM；软引用：被软引用指向的对象是熬不过第一次垃圾收集的；虚引用用的很少，主要作用与直接内存，被其引用的对象不能根据引用获取对象，这个对象回收后会收到一个通知。这些都是属于java虚拟机层面的，如果看的很吃力的话，我推荐周志明老师的深入了解JVM书籍，看过后会对JVM有所熟悉，对源码理解会更加透彻。<br>
2. <font color=red>Q：为什么Entry要继承弱引用？内存为什么泄露？</font><br>
   A：首先ThreadLocal有一个好处就是，不管何时何地，只要我能够获取到ThreadLocal引用，我能够跨两个的方法进行存取值，而且不需要传递任何参数，但是有一个前提条件，就是这个线程必须是同一条线程。但是选取什么作为key就很关键，设想一下，假设以String类型作为key，两个方法，一个方法通过这个key存值，另一个方法越过这个key去取值，两个方法之间必定会传递这个String类型的key，所以ThreadLocal以自己为key，就省去了很多麻烦的操作。刚才我说过，java引用强度四种，其实在这里我们可以排除两个引用来设计ThreadLocal，一个是虚引用，哪怕对象不被回收，也不能访问对象，这肯定是不可以的，另一个是软引用，因为这种引用过于极端，内存够时，内存泄露(这里还没有讲为什么会内存泄露，读者可以回过头理解这句话)，内存不够时，被回收，但是这存在赌的概念，一旦回收了内存还不够内存会溢出的，所以也不在设计范围内。下面还剩下强引用和软引用，强引用不用继承任何类，普通的类都是强引用。首先读者要知道一点，内存泄露是多个引用指向同一个对象，由于某个操作导致无用的对象或者是无法利用的对象被强引用指向而一直存活，从而得不到回收。我们通过ThreadLocal外部的实例引用（t1）设置值（如下代码），从下面代码可以知道ThreadLocal tl = new ThreadLocal();其t1作为外部引用，Entry.key作为内部引用，这两个引用指向同一个ThreadLocal实例，假设用强引用设计，一旦外部的引用tl = null;由于我取值是通过tl.get()去取的，但是我外部的key已经被我设置成null了，我不可能null.get()吧，因此我无法通过外部引用访问到当前线程的ThreadLocalMap的存储数据了，这样的数据就会以[key,value]的形式“胎死腹中”，此时会形成两个问题，第一，内存泄露了，第二，由于无用的数据和有用的数据都是以[key,value]的形式存在，无法判断哪个Entry是垃圾对象。那我们回过头来看，用软引用能不能解决这两个问题。如果使用软引用，t1 = null，同样以前存放的数据依旧无法取出来，但是由于内部引用key是弱引用，在第一次垃圾收集就会被回收，所以以[null,value]存放在ThreadLocalMap中，相比于强引用的[key,value]形式，内存虽然还泄露，但是保证了key的部分内存是不泄露的，value和Entry为什么不被弱引用指向，value可能存放任何对象，存储的值外界基本上不存在强引用指向，岂能随便回收，这是其一，其二，随便继承引用类具有侵入性，导致你无法继承其他类。由于以[null,value]存放，对于造成内存泄露的对象我们就有了判断，只要key为null的肯定都是无用对象，解析到这里，读者应该也知道为什么要用弱引用了吧。<br>
3. <font color=red>Q：如何防止内存泄露</font><br>
   A：让ThreadLocal外部的引用直接或者间接处于GC ROOT集合中，从而形成一个可达性，让其无法被回收，ThreadLocal的set，get，remove方法对泄露的对象都有clean操作。

    ```java
    // t1是外部引用
    ThreadLocal tl = new ThreadLocal();

    // 根据源码得知
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            // 由此进入
            createMap(t, value);
    }

    void createMap(Thread t, T firstValue) {
        // 为此线程创建一个ThreadLocalMap，这边传入了一个this当前对象引用，就是此ThreadLocal的引用
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

    // ThreadLocalMap的构造函数
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        // Entry数组，初始容量是16
        table = new Entry[INITIAL_CAPACITY];
        // 根据ThreadLocal的hashcode计算在table的索引值
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        // 新建一个Entry将其放入到table所对应的位置，这里面firstKey为this
        table[i] = new Entry(firstKey, firstValue);
        // 元素为1
        size = 1;
        // 设置数据的阈值，方法比较简单threshold = len * 2 / 3为10
        setThreshold(INITIAL_CAPACITY);
    }

    Entry(ThreadLocal<?> k, Object v) {
        // 底层是referent的弱引用
        super(k);
        value = v;
    }
    ```

### Entry的类结构
```java
// Entry继承了弱引用，ThreadLocal类型的key为弱引用
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```


**如果对于我上面的分析都能看得懂的话，下面我们一起探究ThreadLocal是如何处理内存泄露的**
<br>
★符号所对应的号码我在下面有做详细讲解
1. 先从set方法开始

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 当map存在时
        map.set(this, value);
    else
        // 初始化map源码我在上面已经讲过了
        createMap(t, value);
}

// key是this对象
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.
    // 获取table
    Entry[] tab = table;
    // 获取其长度
    int len = tab.length;
    // 通过ThreadLocal的HashCode计算该ThreadLocal处于tab的索引值
    int i = key.threadLocalHashCode & (len-1);

    // 获取这个位置的对象，如果有对象，进行判断，判断不符合条件，取下一个下标的对象，知道取到null为止
    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        // 获取这个位置所对应对象的key，由于是弱引用，有可能有值，有可能为null
        ThreadLocal<?> k = e.get();
        // 由于ThreadLocal外界引用和内部引用指向同一个对象，判断相等只需要看引用相等就行了，如果看过HashMap源码的朋友，没有必要以HashMap的判断去衡量key的相等。
        if (k == key) {
            // 如果key一样，是更新操作，只需要覆盖旧值
            e.value = value;
            // 覆盖成功，方法没有必要走下去了
            return;
        }
        // ★⑴
        if (k == null) {
            // 如果k为null，说明其他ThreadLocal被回收了，咱先不看这个方法里怎么处置，从上下文可以推测出
            // 会将这个位置的元素清理，然后在这个位置放入我们的值，走完这个方法return了发现没，而且还传了
            // 三个可以考究的参数
            // ★⑵
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 因为上面的for循环进行next下标迭代，迭代的元素key比较不一样，中途也没有内存泄露的对象，
    // 则上面是迭代到null为止，所以直接在这个位置存储值
    // 而且如果计算key.threadLocalHashCode & (len-1)位置为null，直接不走for循环，这个时候真的是放在自己原本要放的位置
    tab[i] = new Entry(key, value);
    // 元素+1
    int sz = ++size;
    // ★⑽ 看看是否需要rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

// 看到这心态都崩了呀，这啥啊
// 首先传入了三个参数，当前ThreadLocal对象，设置的值和内存泄露的下标
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // slotToExpunge记录内存泄露下标
    int slotToExpunge = staleSlot;
    // 下标前移进行迭代,直到null为止
    for (int i = prevIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = prevIndex(i, len))
        // 如果发现内存泄露对象 ★⑶
        if (e.get() == null)
            // 记录前一个内存泄露对象的位置最小下标
            slotToExpunge = i;

    // 从内存泄露对象位置后移动
    for (int i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 不排除我先遇到内蕴泄露进入这个方法，下标后移有可能是update操作，直到遇到null为止，解读★⑷
        if (k == key) {
            // 此时是update操作，肯定return
            e.value = value;
            // 将内存泄露的对象赋到i处
            tab[i] = tab[staleSlot];
            // 将更新后的数据移到staleSlot处
            tab[staleSlot] = e;
            // 说明之前迭代的key不为null，不然走下面的if，slotToExpunge就比staleSlot大了（不得不配服这些写源码的人，咱们只配看）此时的slotToExpunge == staleSlot说明前移没有找到内存泄露对象，后移没有找到key为null的情况，即内存泄露对象
            if (slotToExpunge == staleSlot)
                // 由于tab[staleSlot]内存泄露对象移到i处，所以此时最小内存泄露是i
                slotToExpunge = i;
            // ★⑺
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }
        
        // slotToExpunge == staleSlot的情况我在★⑶讲解过，所以此时slotToExpunge是迭代null位置为止内存泄露的最小下标★⑸
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 假设是第一次放值，能进入本方法说明我要把这个内存泄露的对象给替换掉
    // If key not found, put new entry in stale slot
    // 将值清理
    tab[staleSlot].value = null;
    // 放入我们的值
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    // 对于slotToExpunge，本方法对新建对他有两处改动，一处是前移最小内存泄露下标，一处是后移最小内存泄露下标
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}

// 将内存泄露下标传入
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    // 由于泄露了，value为null
    tab[staleSlot].value = null;
    // 该位置的entry也为null，一定要先保证value为null
    tab[staleSlot] = null;
    // 元素个数-1
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    // 有可能replaceStaleEntry方法右移没有到null，就进入本方法，迭代完
    for (i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            // ★⑹将staleSlot位置往后的到下一个null为止的所有内存泄露对象清理
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // 没有内存泄露的对象计算索引值
            int h = k.threadLocalHashCode & (len - 1);
            // 如果不符合迭代的下标
            if (h != i) {
                // 将这个位置为null，重新找自己的位置
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                // 如果自己的位置被别的ThreadLoca占着，只能再次右移找自己的位置
                while (tab[h] != null)
                    h = nextIndex(h, len);
                // ★⑻
                tab[h] = e;
            }
        }
    }
    // 传出下一个为null的下标
    return i;
}

private void rehash() {
    expungeStaleEntries();
    // 上面整个方式是对整个数组一次大清理，执行完之后可以确保是没有内存泄露的
    // 整个设计时可以理解的，确保我要不要扩容，我先把整个数组清理一遍

    // Use lower threshold for doubling to avoid hysteresis
    // 使用较低的加倍阈值以避免滞后，主要是快要满了，那我就resize，说来有点意思
    // threshold并不是真正的阈值threshold - threshold / 4 才是
    if (size >= threshold - threshold / 4)
        resize();
}

// 扩容 Double the capacity of the table.
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    // 扩容两倍，不解的是为什么不用左移，我估计是本来就是线性表，本来就慢，不差这点慢效率
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                //如果发现内存泄露，只需要把value丢掉，entry为什么不丢，整个旧数组都要丢掉，肯定是包含entry
                // 不管value外界有没有强引用指向，我还是将其清理
                // expungeStaleEntries将整个数组清理了，但是我这边还是去判断是否内存泄露
                // 整个代码的健壮性肉眼可见。
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                // 记录元素个数
                count++;
            }
        }
    }
    // 整个方法没什么难的
    setThreshold(newLen);
    size = count;
    table = newTab;
}

private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    // 遍历整个数组
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            // 一旦发现内存泄露对象，就整理一次，整理之后我可以确保本次下标被清理了，至于对象移动这个我们不管，因为这是遍历整个数组。
            expungeStaleEntry(j);
    }
}



// 清理其他的Slot（两个null之间存在元素称之为Slot，个人理解）
private boolean cleanSomeSlots(int i, int n) {
    // 是否
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        // 迭代下一个元素
        i = nextIndex(i, len);
        Entry e = tab[i];
        // 如果再迭代过程中发现内存泄露对象
        if (e != null && e.get() == null) {
            // 将数组长度重新赋给n，让其有跟多次数迭代
            n = len;
            removed = true;
            // ★⑼
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}

// 下标前移
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}

// 下标后移
// 获取下一个下标，大家也可以认为这是数组下标的轮询算法，当然实现轮询有好几种，比如说ribbon负载均衡中的轮询，就是采用原子类AtomicInteger来保证原子性，+1取余的这种方式进行轮询。
private static int nextIndex(int i, int len) {
    // 如果超过长度回到下标0嘛
    return ((i + 1 < len) ? i + 1 : 0);
}
```

<font color=red>★⑴</font>有些人有可能会疑问了，该位置的key为什么会为null，这我先得从ThreadLocalMap来说起，但是我又不得不提HashMap来做一个对比，这两个有一个共同的数据结构就是数组，HashMap是Node[]，ThreadLocalMap是Entry[]，但是HashMap中Node类是一个链表结构，而ThreadLocalMap不是，所以HashMap如果多个key计算的索引值一样，就会形成链表，采取一个纵向设计，但是由于ThreadLocal存在内存泄露风险，采取一个横向设计，所以设计ThreadLocalMap的数组元素只能放一个，但是不可避免两个ThreadLocal对象计算的索引值一样，为了保证不去覆盖别的ThreadLocal在本线程中存储的值，只好不停迭代下一个元素，来寻找自己存放的位置，这句话可以衍生出两个意思,第一就是在时间顺序上之前的ThreadLocal对象计算出的索引值和当前ThreadLocal对象计算的索引值一样，从而占着位置，第二个就是先前的ThreadLocal对象其位置被其他ThreadLocal占了，迭代到你的位置，从而占领你的位置。所以int i = key.threadLocalHashCode & (len-1);计算出的索引值不一定就放在这个位置，至于key为null的情况，我在★⑷中举例子说明了，可以参考。<br>
<font color=red>★⑵</font>这边大家可以进行思考，如果我第一次放ThreadLocal对象，根据set方法来推测下标范围，下标为[key.threadLocalHashCode & (len-1),后移迭代Entry数组第一个为null的下标]<br>
<font color=red>★⑶</font>这里有一个细节，从这个内存泄露下标往前移，[key.threadLocalHashCode & (len-1)，staleSlot]之间肯定不会存在null，和内存泄露的对象，这是因为set方法中的for循环，大家可以仔细思考一下，所以前移的位置的下标值一定小于key.threadLocalHashCode & (len-1)，slotToExpunge的值就是记录就是下标前移内存泄露对象的下标位置，位置范围是(前移迭代Entry数组第一个为null的下标，key.threadLocalHashCode & (len-1))。但是slotToExpunge = staleSlot也是有可能的，两种情况，从key.threadLocalHashCode & (len-1)前移到null为止之间的位置不存在内存泄露的对象，也有可能key.threadLocalHashCode & (len-1)前面一个就是null，不迭代。<br>
<font color=red>★⑷</font>这里有可能有人就会担心了，ThreadLocal通过计算出索引值，如果被占用，下标后移探测位置直到null为止，以null作为一个“沟”，我感觉用“沟”这个字来形容很恰当。会不会存在null值的下标以后会有这个ThreadLocal引用，从而无法覆盖并生成一个元素，衍生出Entry数组中存在两个相同ThreadLocal为key的存储数据。这种思考是多余的，因为从源码阅读过程中可以发现，从key.threadLocalHashCode & (len-1)计算出索引值开始，一点点后移去探测，如果中间没有其他ThreadLocal内存泄露，直到null，将这个位置占领，如果是其他ThreadLocal内存泄露，清理它并占用这个位置（也有可能是更新操作），所以从key.threadLocalHashCode & (len-1)到探测存储位置的这段元素中有可能[key,value]形式也有可能[null,value]，但绝不可能是null。所以说如果在这中间有[null,value]形式，<font color=red>这种情况是当我第一次放值的时候，由于计算的索引值被人占了，导致我往后迭代，迭代的过程中有可能在泄露对象的地方新建元素，也有可能在后移第一个null位置新建元素，但是由于前面的其他的ThreadLocal对象在我新建这个元素之后外界的强引用消失，导致内存泄露(key为null)，当我第二次放值的时候，在内存泄露的位置进入replaceStaleEntry方法，在这个方法下标后移去查找是否要更新是有必要的。</font><br>
<font color=red>★⑸</font>此时这个判断还是很多人还是有疑问的，从staleSlot后移，记录第一个内存泄露元素下标，为什么是第一个，slotToExpunge = i嘛，赋了一次值后slotToExpunge肯定是等于staleSlot+1的。但是我说slotToExpunge为最小内存泄露下标恐怕很多人不服，为什么，难道staleSlot位置不是内存泄露嘛，replaceStaleEntry方法不就是因为这个位置内存泄露而进入的吗。但是我们要知道一点，对于新建操作，staleSlot下标是要被我们新建的元素替换掉的（不过看源码更新也是把这个位置替换掉），替换之后slotToExpunge难道不是内存泄露的最小下标吗<br>
<font color=red>★⑹</font>此时大家已经感觉到staleSlot是你的ThreadLocal替换原来泄露的对象之后在你左边为null为止至你右边为null为止的最小下标，其实在这里说最小下标不是很合适，因为nextIndex它是右移轮询算法，有可能是从尾轮询到头，暂且称之为两个null之间的最左侧位置。由于在replaceStaleEntry方法中求得两个null之间的最左侧内存泄露的位置，所以expungeStaleEntry方法只需要向右移至下一个null为止，清理内存对象即可，保证左侧null和右侧null之间没有内存泄露对象。<br>
<font color=red>★⑺</font>cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);是在replaceStaleEntry方法中两处调用，一个是新建操作，一个是更新操作，但是都有一定的条件，首先我们先看新建，调用的前提条件是slotToExpunge != staleSlot，而达到这个条件有两处，下标前移到前一个null为止，中间存在内存泄露对象或者是，下标后移slotToExpunge被改变，但是被改变又存在k == null && slotToExpunge == staleSlot的条件，这就意味着前移不存在内存泄露对象，后移存在，如果从staleSlot位置前移后移到null这段过程中没有任何内存泄露对象，此时slotToExpunge == staleSlot是唯一内存泄露的地方，当前ThreadLocal清理并占领即可，这段内存就没有必要clean了，这是新建操作的解读。下面看更改，更改有三处都有对slotToExpunge修改，包含新建的两个，还有一个就是在更改操作中，从源码可看，当前面两个操作没有对slotToExpunge更改时，也就是说前移没有内存泄露对象，后移也没有泄露对象，直到我发现我是更改操作，此时staleSlot是内存泄露位置，更改肯定不会像新建一样去弥补这块内存泄露，它有自己占的位置，所以呢就将自己的元素移到staleSlot位置，迭代的i下标就成了新的泄露最左侧位置了。<br>
<font color=red>★⑻</font>这里由于清理内存泄露而让一些没有内存泄露对象重新寻找索引位置，不过这一次寻找和以往的寻找有点不一致，以往不管是新建还是更改探测索引位置，都是右移到null为止，如果中间发现内存泄露对象，将它替换掉。而这次移动元素只是右移探测到null为止，然后就地“安营扎寨”。对途中所有的内存泄露对象是不管的。为什么呢，expungeStaleEntry方法就是本身就是对内存泄露对象的清理，如果在因为移动元素途中在尝试对内存泄露对象进行清理，势必要写一个类似于递归的操作，如果一直这样没完没了的下去，恐怕离栈溢出不远了。还有一个就是未造成内存泄露的对象重新计算索引值，是害怕前面有些内存泄露对象被清理，形成null，到时候get方法或者再次赋值的时候被这个null阻塞从而再生成新元素。<br>
<font color=red>★⑼</font>expungeStaleEntry方法将迭代到为null的下标传进cleanSomeSlots方法，cleanSomeSlots在后移进行查找，这种方式我认为是“跨slot”操作，这里是我个人理解，就是前后为null之间算一个slot，而expungeStaleEntry迭代到后一个null下标，我如果从这个下标做处理，就是跨到下一个前后为null之间的slot了，由于我是跨slot加后移，所以肯定会遇到该域中第一个内存泄露对象的，符合expungeStaleEntry方法。用了一个do-while做迭代，条件是(n >>>= 1) != 0，如果n是16的话，就可以迭代5次，2的四次方是5次，那2的n次方就是n+1次。但是如果在迭代过程中发现有内存泄露对象时，继续expungeStaleEntry方法对当前泄露对象的域进行clean，然后对未造成内存泄露的对象重新找地址，返回下一个null下标，而且table.length重新赋值给n，这样给人感觉次数又重新充满了，那是因为发现内存泄露问题。次数肯定是不能降的。假设数组长度是16，则允许我迭代五次，如果连续遇到五个没有内存泄露的对象，则跳出do-while循环，假设我第6个元素是内存泄露对象，就没办法对他进行清理了，这种算法是一种优化算法，并没有把整个数组全部检查一遍，所以呢cleanSomeSlots方法真的是和这个方法名一样，只能清理一些slot的内存泄露，不能清理全部，不然为什么不叫cleanAllSlots呢。<br>
<font color=red>★⑽</font>尝试去clean元素，如果说至少clean一个元素，就没有必要rehash了，因为增一个减至少一个，数量肯定不会增加。如果没有clean至少一个元素，由于我添加一个新的元素，就得查看有没有超过阈值，超过了就得rehash。

**以上是set方法的讲解，内容也很多，理解起来并不容易，有很多编码细节和算法值得我们去研究，当然上面这么多理解，也仅仅是我个人的理解，希望对你有帮助。**

2. get方法
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 查找是否有这个元素
        ThreadLocalMap.Entry e = map.getEntry(this);

        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // map为null或者是map不为null且元素不存在
    return setInitialValue();
}

// 其他情况
private T setInitialValue() {
    // get也是可以懒加载map的，
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    // 返回null，这方法太简单，看下面方法
    return value;
}

// 默认初始值是null
protected T initialValue() {
    return null;
}

// 查找元素
private Entry getEntry(ThreadLocal<?> key) {
    // 计算索引值
    int i = key.threadLocalHashCode & (table.length - 1);
    // 获取元素
    Entry e = table[i];
    // 元素不为null，并且就在这个位置，直接返回，否则走else
    if (e != null && e.get() == key)
        return e;
    else
        // miss
        return getEntryAfterMiss(key, i, e);
}

// 如果e等于null，直接返回，也就是说没找到元素，解读一下★⑴
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        // 获取当前迭代的对象的key，下面判断key是否相等，在传进来的e就达到key相等是
        // 永远不可能进getEntryAfterMiss方法的，因为getEntry的if条件
        ThreadLocal<?> k = e.get();
        if (k == key)
            // 找到直接返回
            return e;
        if (k == null)
           // ★⑵
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

<font color=red>★⑴</font>其实从前面的分析讲解中发现，不管是新建元素还是移动元素（replaceStaleEntry中的更新操作或者是expungeStaleEntry中的移动元素），都会尝试去占领属于自己的位置。所以当前ThreadLocal计算出的下标在实际存放位置的前面或者下标一致都是有可能的。如果说计算出的下标没有值，在当时新建或者移动时候就肯定会放在这个位置，这个位置在get时候不会为null了，所以说这个位置只能有值，有可能是当前ThreadLocal存放的数据，有可能被其他ThreadLocal占着，至于其他ThreadLocal内存泄露还要做出判断和处理。所以说如果这个位置为null了，说明map里面真的没存过当前ThreadLocal对象，也没有必要迭代寻找，这是ThreadLocal存放的规则。所以写代码的时候也一定要钻规则中隐含的条件作为判断，这才是最高境界。<br>
<font color=red>★⑵</font>此处调用expungeStaleEntry方法，和replaceStaleEntry调用不太类似，replaceStaleEntry将整个slot清理了，因为计算了slot中最小泄露位置，而getEntryAfterMiss不是，我并没有计算key.threadLocalHashCode & (table.length - 1)前边的泄露下标。getEntryAfterMiss之后尝试再次获取tab[i]，如果是null，说明没有，其实这边和★⑴是有异曲同工之妙的。单单看i下标，这是key.threadLocalHashCode & (table.length - 1)往后迭代得到的，条件是中途不能有null，假设i下标泄露了，expungeStaleEntry会将此处清理成null。如果后面存在我们要获取的元素，当前ThreadLocal势必会在expungeStaleEntry中重新计算索引值位置，但是此时getEntryAfterMiss方法从threadLocalHashCode & (table.length - 1)迭代到i位置，就是因为中途没有null，所以说真的后面有存储值，也有可能重新放在i位置，我在这里也仅仅说是有可能，还有可能被其他元素占了，导致我要找的元素放到i下标后面，但是有一点完全保证，如果当前ThreadLocal真的有值，i处决不可能为null，换句话说就是i处为null，绝对没值，没必要继续查找，直接返回null就行。还有就是expungeStaleEntry方法我没有细讲，但是我是要说一下，由于这个方法是一个slot中指定内存泄露对象下标往后迭代清理，但是一些元素边清理，边重新算位置，清理的元素大于要算位置的元素，肯定会出现很多null位置，有可能一个slot会被null切割变成几个slot，但是不要担心，所有的ThreadLocal的key.threadLocalHashCode & (table.length - 1)肯定还是会在一个solt之间，你实际存储的位置肯定也会在相同的solt中，下标范围是[key.threadLocalHashCode & (table.length - 1)，下一个null下标]。

**老实说，ThreadLocal的源码分析比HashMap源码分析起来更加吃力一些，需要强大的逻辑和一定的想象空间才能理会到我上面详细讲的每一点，像这种源码分析代入感特别强，其实我现在总结的理解，在分析的时候理解的太深，当我一段时间不看了，做其他事了，我甚至会怀疑这文档是不是我写的，再次理解也需要依靠一些源码去支持。所以说多看源码，别人大神写的代码的细节点我们要领会到，这样以后写一些复杂的功能时候会有所帮助。**

3. remove方法
```java
// 将当前ThreadLocal对象移除
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

// 上面方法简单，咱看这个方法
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    // 迭代
    for (Entry e = tab[i];
            e != null;
            e = tab[i = nextIndex(i, len)]) {
        // key相同
        if (e.get() == key) {
            // 清空弱引用
            e.clear();
            // 从这个位置开始clean，类似于这个位置泄露，哈哈，其实没有啦，是e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

**这个源码的分析就是这样，理解起来不容易，如果都能吃透的话，那就很厉害了。**
