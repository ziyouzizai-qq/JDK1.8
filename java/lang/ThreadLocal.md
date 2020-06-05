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

**首先从Entry的类结构我们来对问题2和问题2做一个简单的回答**
1. Q：什么是弱引用？
   A：java里面的引用分为强软弱虚四个引用强度，强引用，我们平时new的对象，没有引用指向才能被回收；软引用，当整个堆内存不够时，java虚拟机会尝试回收掉软引用的对象，如果回收了内存还不够，报OOM；软引用：被软引用指向的对象是熬不过第一次垃圾收集的；虚引用用的很少，主要作用与直接内存，被其引用的对象不能根据引用获取对象，这个对象回收后会收到一个通知。这些都是属于java虚拟机层面的，如果看的很吃力的话，我推荐周志明老师的深入了解JVM书籍，看过后会对JVM有所熟悉，对源码理解会更加透彻。
2. Q：为什么Entry要继承弱引用？内存为什么泄露？
   A：首先ThreadLocal有一个好处就是，不管何时何地，只要我能够获取到ThreadLocal引用，我能够跨两个的方法进行存取值，而且不需要传递任何参数，但是有一个前提条件，就是这个线程必须是同一条线程。但是选取什么作为key就很关键，设想一下，假设以String类型作为key，两个方法，一个方法通过这个key存值，另一个方法越过这个key去取值，两个方法之间必定会传递这个String类型的key，所以ThreadLocal以自己为key，就省去了很多麻烦的操作。刚才我说过，java引用强度四种，其实在这里我们可以排除两个引用来设计ThreadLocal，一个是虚引用，哪怕对象不被回收，也不能访问对象，这肯定是不可以的，另一个是软引用，因为这种引用过于极端，内存够时，内存泄露(这里还没有讲为什么会内存泄露，读者可以回过头理解这句话)，内存不够时，被回收，但是这存在赌的概念，一旦回收了内存还不够内存会溢出的，所以也不在设计范围内。下面还剩下强引用和软引用，强引用不用继承任何类，普通的类都是强引用。首先读者要知道一点，内存泄露是多个引用指向同一个对象，由于某个操作导致无用的对象或者是无法利用的对象被强引用指向而一直存活，从而得不到回收。我们通过ThreadLocal外部的实例引用（t1）设置值（如下代码），从下面代码可以知道ThreadLocal tl = new ThreadLocal();其t1作为外部引用，Entry.key作为内部引用，这两个引用指向同一个ThreadLocal实例，假设用强引用设计，一旦外部的引用tl = null;由于我取值是通过tl.get()去取的，但是我外部的key已经被我设置成null了，我不可能null.get()吧，因此我无法通过外部引用访问到当前线程的ThreadLocalMap的存储数据了，这样的数据就会以[key,value]的形式“胎死腹中”，此时会形成两个问题，第一，内存泄露了，第二，由于无用的数据和有用的数据都是以[key,value]的形式存在，无法判断哪个Entry是垃圾对象。那我们回过头来看，用软引用能不能解决这两个问题。如果使用软引用，t1 = null，同样以前存放的数据依旧无法取出来，但是由于内部引用key是弱引用，在第一次垃圾收集就会被回收，所以以[null,value]存放在ThreadLocalMap中，相比于强引用的[key,value]形式，内存虽然还泄露，但是保证了key的部分内存是不泄露的，value和Entry为什么不被弱引用指向。因为他们没有强引用指向，垃圾收集数据会丢失。由于以[null,value]存放，对于造成内存泄露的对象我们就有了判断，只要key为null的肯定都是无用对象，解析到这里，读者应该也知道为什么要用弱引用了吧。
3. Q：如何防止内存泄露
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

★⑴有些人有可能会疑问了，该位置的key为什么会为null，这我先得从ThreadLocalMap来说起，但是我又不得不提HashMap来做一个对比，这两个有一个共同的数据结构就是数组，HashMap是Node[]，ThreadLocalMap是Entry[]，但是HashMap中Node类是一个链表结构，而ThreadLocalMap不是，所以HashMap如果多个key计算的索引值一样，就会形成链表，采取一个纵向设计，但是由于ThreadLocal存在内存泄露风险，采取一个横向设计，所以设计ThreadLocalMap的数组元素只能放一个，但是不可避免两个ThreadLocal对象计算的索引值一样，为了保证不去覆盖别的ThreadLocal在本线程中存储的值，只好不停迭代下一个元素，来寻找自己存放的位置，这句话可以衍生出两个意思,第一就是在时间顺序上之前的ThreadLocal对象计算出的索引值和当前ThreadLocal对象计算的索引值一样，从而占着位置，第二个就是先前的ThreadLocal对象其位置被其他ThreadLocal占了，迭代到你的位置，从而占领你的位置。所以int i = key.threadLocalHashCode & (len-1);计算出的索引值不一定就放在这个位置，所以这个位置如果说被其他ThreadLocal对象占着，然后他又失去外界的强引用，k就有可能是null。
★⑵这边大家可以进行思考，如果我第一次放ThreadLocal对象，根据set方法来推测下标范围，下标为[key.threadLocalHashCode & (len-1),后移迭代Entry数组第一个为null的下标]
★⑶这里有一个细节，从这个内存泄露下标往前移，[key.threadLocalHashCode & (len-1)，staleSlot]之间肯定不会存在null，和内存泄露的对象，这是因为set方法中的for循环，大家可以仔细思考一下，所以前移的位置的下标值一定小于key.threadLocalHashCode & (len-1)，slotToExpunge的值就是记录就是下标前移内存泄露对象的下标位置，位置范围是(前移迭代Entry数组第一个为null的下标，key.threadLocalHashCode & (len-1))。但是slotToExpunge = staleSlot也是有可能的，两种情况，从key.threadLocalHashCode & (len-1)前移到null为止之间的位置不存在内存泄露的对象，也有可能key.threadLocalHashCode & (len-1)前面一个就是null，不迭代。
★⑷这里有可能有人就会担心了，ThreadLocal通过计算出索引值，如果被占用，下标后移探测位置直到null为止，以null作为一个“沟”，我感觉用“沟”这个字来形容很恰当。会不会存在null值的下标以后会有这个ThreadLocal引用，从而无法覆盖并生成一个元素，衍生出Entry数组中存在两个相同ThreadLocal为key的存储数据。这种思考是多余的，因为从源码阅读过程中可以发现，从key.threadLocalHashCode & (len-1)计算出索引值开始，一点点后移去探测，如果中间没有其他ThreadLocal内存泄露，直到null，将这个位置占领，如果是其他ThreadLocal内存泄露，清理它并占用这个位置（也有可能是更新操作），所以从key.threadLocalHashCode & (len-1)到探测存储位置的这段元素中有可能[key,value]形式也有可能[null,value]，但绝不可能是null，在这中间有[null,value]形式，这种情况是当我第一次放值的时候，由于计算的索引值被人占了，导致我往后迭代，迭代的过程中有可能在泄露对象的地方新建元素，也有可能在后移第一个null位置新建元素，但是由于前面的其他的ThreadLocal对象在我新建这个元素之后外界的强引用消失，导致内存泄露，当我第二次放值的时候，在内存泄露的位置进入replaceStaleEntry方法，在这个方法下标后移去查找是否是更新操作是有必要的。
★⑸此时这个判断还是很多人还是有疑问的，从staleSlot后移，记录第一个内存泄露元素下标，为什么是第一个，slotToExpunge = i嘛，赋了一次值后slotToExpunge肯定是等于staleSlot+1的。但是我说slotToExpunge为最小内存泄露下标恐怕很多人不服，为什么，难道staleSlot位置不是内存泄露嘛，replaceStaleEntry方法不就是因为这个位置内存泄露而进入的吗。但是我们要知道一点，对于新建操作，staleSlot下标是要被我们新建的元素替换掉的（不过看源码更新也是把这个位置替换掉），替换之后slotToExpunge难道不是内存泄露的最小下标吗
