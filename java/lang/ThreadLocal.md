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