# AbstractQueuedSynchronizer(简称AQS)源码解析

### 简介：AbstractQueuedSynchronizer抽象同步队列简称AQS，juc包下中锁的底层实现都是使用的AQS。这次源码解析我不会直接对AbstractQueuedSynchronizer进行单独解析，我会结合ReentrantLock一起解析，所以说这次还需要建立 ReentrantLock.md，一起来理解什么是AQS？

<hr />

## AQS的内部类Node

**Node**

```java
static final class Node {
    // 下面两个为一类标记
    // 用来标记该线程是获取共享资源时倍阻塞挂起后进入AQS队列，就是个标记
    static final Node SHARED = new Node();
    // 和上面SHARED一样起标记作用，目的是获取独占资源时阻塞挂起至AQS队列
    static final Node EXCLUSIVE = null;

    // 下面四个为一类标记
    // 线程被取消标记
    static final int CANCELLED = 1;
    // 线程需要唤醒标记
    static final int SIGNAL = -1;
    // 线程在条件队列里面等待标记
    static final int CONDITION = -2;
    // 释放共享资源时需要通知其他节点标记
    static final int PROPAGATE = -3;

    // 记录当前线程等待状态
    volatile int waitStatus;

    // 类似于LinkedList
    // 前节点
    volatile Node prev;
    // 后节点
    volatile Node next;
    // 存放进入AQS队列里面的线程
    volatile Thread thread;
    // 独占和共享的标志
    Node nextWaiter;

    final boolean isShared() {
        return this.nextWaiter == SHARED;
    }

    final Node predecessor() throws NullPointerException {
        // 获取前节点，空指针检查，并抛出，利用抛出异常来终止线程
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

#### AQS的实例方法(独占锁获取和释放)，下面是独占方式下获取资源和释放资源时使用的方法，可以和ReentrantLock联系在一起，里面用了大量的接口调用和模板方法设计模式。
```java
// 独占锁获取
public final void acquire(int arg) {
    // tryAcquire其实在AQS中是一个模板方法形式存在，下面的tryRelease方法也是，很显然，这些都需要子类实现。
    // 以ReentrantLock为例来获取锁，请移步至ReentrantLock.md文档中ReentrantLock的实例方法lock()
    // ★⑴详情
    if (!tryAcquire(arg) &&
        // 非占用锁的线程进入等待队列
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// 如果当前线程被中断过，之后又处于第一个线程node争抢锁成功后返回，就进入selfInterrupt
// 方法很简单
static void selfInterrupt() {
    // 就是将当前线程的中断标志重新设上。
    Thread.currentThread().interrupt();
}

// 独占锁释放
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```
<hr />

#### unsafe类
**AQS中关于unsafe类的使用**

**简单说一下unsafe类，JDK的rt.jar包中unsafe类提供了硬件级别的原子性操作，unsafe类中都是本地方法，native修饰，通过JNI的方式直接访问底层c库，是官方开的一个“后门”，功能非常强大，但是在JDK8中我们是没有权限使用这个类的。至于为什么没有权限，下面会说明。**
```java
// expect期望值，update更新值
protected final boolean compareAndSetState(int expect, int update) {
    // 通过CAS(比较并交换)的方式进行原子性更新操作，绝对的线程安全。
    // 具体用法是如果this（当前对象）中内存偏移量stateOffset的变量值是expect，这使用新的值
    // 更换旧的值，成功为true。
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}

// 通过CAS操作对AQS的尾节点tail进行交换
private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}

// 通过CAS操作对AQS的头节点head进行交换
private final boolean compareAndSetHead(Node update) {
    return unsafe.compareAndSwapObject(this, headOffset, null, update);
}

// AQS中获取Unsafe实例
private static final Unsafe unsafe = Unsafe.getUnsafe();

private static final long stateOffset;
private static final long headOffset;
private static final long tailOffset;
private static final long waitStatusOffset;
private static final long nextOffset;

static {
    try {
        // 类加载时将AQS中state，head，tail，waitStatus，next通过反射取出字段，利用unsafe
        // 计算出各个属性的内存偏移量，为CAS做准备
        stateOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
        headOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
        tailOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
        waitStatusOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("waitStatus"));
        nextOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("next"));

    } catch (Exception ex) { throw new Error(ex); }
}

```
**JDK8中Unsafe类我们是无法使用，请看Unsafe类源码**
**Unsafe类是一个典型的饿汉式的单例模式**

```java
// 单例的常量
private static final Unsafe theUnsafe;
static {
    theUnsafe = new Unsafe();
    // 。。。
}

// 封装构造函数
private Unsafe() {
}

// 对外暴露获取Unsafe的唯一实例，因此，如果通过正规途径获取Unsafe实例必须调用getUnsafe方法
// 但是这个方法会对调用类进行类加载器检查，Unsafe实例根据源码只能被Bootstrap(启动类)加载器加载的类
// 来调用。但是Bootstrap加载器只会加载rt.jar包下的文件，而我们用户(对于官方，我们程序员都是java的用户)来说，
// 我们写的程序都是用的应用类加载器进行加载。
public static Unsafe getUnsafe() {
    // 获取调用类
    Class var0 = Reflection.getCallerClass();
    if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
        // 如果不是启动类加载器，报SecurityException
        throw new SecurityException("Unsafe");
    } else {
        // 如果是，就可以获取theUnsafe，所以说，官方的类是存在于rt.jar包下用启动类加载器加载class，
        // 是存在权限访问Unsafe类的。
        return theUnsafe;
    }
}

// VM.class
public static boolean isSystemDomainLoader(ClassLoader var0) {
    // 判断是否为启动类加载器，由于启动类加载器是用C++写的，这类加载器为null
    return var0 == null;
}
```

**虽然在JDK8之前我们是无法使用强大的Unsafe类，但是在JDK9之后官方在VarHandle类里让我们可以使用，那如果我想在8版本中强行获取Unsafe实例怎么办，可能有人会想到反射，很强大，但是并不推荐。**
<hr />


#### AQS的重要属性

1. state：由于AQS我们都不会直接去使用，都是选用并发包下的锁，所以对于AQS的不同实现来说state所代表的意思是不一样的，就拿ReentrantLock来说，state表示当前线程获取锁的可重入次数。当然，对于像semaphore，ReentrantReadWriteLock等又是不同的意思。
```java
private volatile int state;
/**
 * Returns the current value of synchronization state.
 * This operation has memory semantics of a {@code volatile} read.
 * @return current state value
 */
protected final int getState() {
    return state;
}

/**
 * Sets the value of synchronization state.
 * This operation has memory semantics of a {@code volatile} write.
 * @param newState the new state value
 */
protected final void setState(int newState) {
    state = newState;
}
```

<font color=red>★⑴</font>这里由于ReentrantLock调用了lock方法而进来。前提是通过CAS操作期望把state从0变为1失败而进来，这说明了ReentrantLock对象已经被线程占有。但是也有可能ReentrantLock是被自己占有着，对于可重入锁来说，这种情况也必须要考虑。ReentrantLock是一个可重入锁，每重入一次，state就+1，这也是为什么ReentrantLock的lock和unLock方法一定要成对使用。这边当CAS失败后还会尝试调用tryAcquire方法，如果在此期间其他线程释放锁，当前线程因此而获得锁，或者是锁的拥有者是自己而重入，就不需要进入等待队列，因为这两种情况都是返回true嘛，再利用&&的短路机制。否则非占有锁的线程就进入等待队列。这里我再说一下，如果有人能走到这里，我再灌输一点鸡汤，不管是AQS，还是Sync下的两个子类，这两段源码看起来很漂亮，里面对抽象，模板方法，重写，继承， 短路机制等玩的特别6，从java语法层面上来讲，这些没有一点难度，但是作者对于这段代码的书写真的是简洁精炼，纯属是基础扎实，经验老到。所以说，要想有这等水平，基础必须要重视，有时候使用最简单的语法写出最有水平的东西出来往往是最难的。<br />


#### addWaiter：进入AQS等待队列方法
```java
private Node addWaiter(Node mode) {
    // 保存当前的线程，并打上挂起标记，如果是独占锁就是Node.EXCLUSIVE
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 获取尾节点作为前驱节点，说明是其他线程已经进入AQS队列
    // 并且要以pred原值通过CAS交换tail的对象
    Node pred = tail;
    if (pred != null) {
        // 设置前驱节点
        node.prev = pred;
        // 之所以用CAS，是因为有可能不止一个线程进入AQS队列，这里势必也会出现线程问题
        // 如果一旦交换成功，说明目前AQS的tail节点为node
        if (compareAndSetTail(pred, node)) {
            // 双向队列，在将前驱节点的后继元素设置成null，必须保证CAS交换成功后才能确定关系，所以在if里面
            pred.next = node;
            // 进入后返回node
            return node;
        }
    }
    // 这种情况有可能是当前AQS队列还没元素，也有可能是AQS队列有元素，只是和其他线程争抢tail失败。
    enq(node);
    return node;
}

private Node enq(final Node node) {
    // 其实这里一看大家如果熟悉的话，典型的自旋。
    for (;;) {
        // 每次都获取tail
        Node t = tail;
        // 如果为null，说明AQS队列为空
        if (t == null) { // Must initialize
            // ★⑵ CAS期望值是null
            // 无参构造来初始化head
            // 这里的头节点是用一个空的node（无线程）设置
            if (compareAndSetHead(new Node()))
                // 头尾一致
                tail = head;
        } else {
            // CAS加for循环组成自旋锁来设置tail
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                // 设置直到成功为止
                // 跳出循环并返回当前线程节点的前驱节点
                return t;
            }
        }
    }
}
```
<font color=red>★⑵</font>这里是通过CAS来设置head的，成功之后再设置tail，设置tail也没有用CAS操作，会不会存在线程安全问题。放心，线程安全是可以保证的。为什么，如果两个线程A和B，A线程发现tail为null，尝试CAS设置head成功后，当想对tail赋值时，处理器分配的时间到了，转而执行B线程，此时的tail还是null，B线程也想通过CAS将head设置了，但是此时A线程已经对head进行设置了，期望值已经不是null，此时B线程的CAS操作就失败了，所以B线程也不会走进if语句中。所以说人家这里写的没有毛病。<br />

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 每一个线程到这里进行自旋
        for (;;) {
            // 获取进入等待队列后的当前线程的节点的前驱节点
            final Node p = node.predecessor();
            // 由于AQS队列用一个空的node作为头，所以对于有线程包裹的node来说，前驱节点不可能为null
            // 尽管在predecessor方法中抛出空指针，但是在调用方并没有对空指针做处理就是因为前驱不可能是null
            // 如果前驱是head，说明当前线程包裹的node是在队列的第二个
            // 再尝试一下去争抢锁，由于已经处于AQS队列，锁的占有者不可能是当前线程，再次尝试只能希望别的线程将锁释放了。
            // 所以说，如果因为争抢锁而进入AQS队列，如果你是第二个节点，因为head是空线程节点，即第一个非空线程节点，你还是有机会再次去争抢锁的。
            if (p == head && tryAcquire(arg)) {
                // 如果一旦当前线程尝试成功了，就将当前节点设置成head，当然也需要为空的node
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 前驱不是head或者是争抢锁失败的情况
            // 所以说一旦当前节点的前驱节点为Node.SIGNAL，就需要把当前线程阻塞挂起
            // 这是需要LockSupport来支持
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 利用LockSupport的park方法挂起并做中断判断
                parkAndCheckInterrupt())
                // 如果当前线程被中断，interrupted为true，等到这个线程node争抢锁成功后返回。
                interrupted = true;
        }
    } finally {
        // ★⑷
        if (failed)
            cancelAcquire(node);
    }
}

// LockSupport类
private final boolean parkAndCheckInterrupt() {
    // 底层还是UNSAFE来操作的
    // LockSupport.park最终把线程交给系统内核进行阻塞，UNSAFE也是直接操作本地方法，
    // 本地方法的方法体是看不见的，因为是用C语言写的。但是具体肯定是通过操作系统去阻塞内核中的线程
    LockSupport.park(this);
    // 当前线程通过LockSupport.park被阻塞挂起，如果在此期间其他线程中断该线程或者是调用unpark。该线程就会被唤醒
    // 判断当前线程是否被中断，被中断返回true，并清理中断标志
    return Thread.interrupted();
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 查看前驱节点的线程等待状态
    int ws = pred.waitStatus;
    // 如果pred是head，ws为默认值0
    // 前驱节点需要被唤醒的情况返回true
    // 由于是自旋。
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    // CANCELLED(线程被取消的情况)的情况
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        // 跳过那些CANCELLED状态的节点，这里具体理解看★⑸
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
         // 其实对于ReentrantLock来说，AQS队列都是到这里，因为所有的node节点waitStatus都是用的默认值0
         // 但是这里主要还是对于前驱元素来讲，则含头不含尾。
         // 但是我理解为将当前线程的node的waitStatus保存到前驱节点上
         // 通过CAS设置前驱节点状态
         // 由于上面对于前驱节点还有结构性变化，还需要通过CAS来保证线程安全，有可能存在waitStatus是CANCELLED的情况。
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private void setHead(Node node) {
    // ★⑶
    head = node;
    // 保证空线程
    node.thread = null;
    // 保证无前驱节点
    node.prev = null;
}
```

<font color=red>★⑶</font>这里也没有做任何的CAS操作来保证原子性，是因为设置head只有第二个节点获取锁成功后才能设置，因此只有第二个进入AQS队列节点的线程有权限设置，给head设置只有一个线程，因此不存在线程安全问题，那这里有人就说了，在enq(final Node node)方法中，不也有设置head吗，请注意enq方法设置的前提，是AQS中没有任何元素时才会设置，很显然这里AQS队列已经存在元素了。在这里setHead方法只对thread和prev进行初始化，是因为在addWaiter中就只对这两个进行设置，因为setHead是因为当前线程为第二个节点并且获得锁，而离开AQS队列，所以对于node的其他属性就是零值，因此setHead对这些属性肯定也不会管的。用白话就是，你是第一个非空线程的节点，尽管你已经来到AQS队列，但是我还是给你一次机会去争抢锁，如果争抢成功，就需要离开队列，所以在没有确定你这次机会成功之前，我只对你维护双向队列关系的属性进行赋值，其他状态一律不设，等你争抢失败后或者不是第一个非空线程的节点在设置为时不晚。<br />

<font color=red>★⑷</font>这里要想进入cancelAcquire方法，failed就必须为true，但是在try语句块中采取的是自旋的方式，唯一退出的方式是处于AQS队列中第二个非空线程节点的线程获取到锁后return，但是在里面failed是被设置成false的。那cancelAcquire还是执行不了，有点感觉是不是人家搞错了，怎么可能人家搞错了，这里是采用抛出异常终止线程从而执行finally块，还记不记得node.predecessor()中抛出一个空指针异常。没错，这里如果一旦抛出NPE，当前线程就会走cancelAcquire，但是不要忘记tryAcquire中也有可能抛出异常new Error("Maximum lock count exceeded")，其实想抛出这两个异常以我现在对源码的理解都挺难的，目前权且理解为是为了保证这两种不正常情况下的健壮性。对于空指针我是这么理解的，肯定不会出现，因为前驱节点为null只有head节点存在，但是head节点是一个哨兵节点(没有任何线程持有的节点，注：这并不是对哨兵节点的解释)，所以说对于线程节点来讲，前驱节点一定存在。那至于state是否存在小于-1的情况，就需要等unlock解析完后再回头看。异常终止线程也可以写个小的demo测试一下。如下：<br />

```java
private static String retStr(String s) throws NullPointerException {
    if (s == null)
        throw new NullPointerException();
    else
        return s;
}

public static void main(String[] args) {
    try {
        retStr(null);
        for (;;) {

        }
    } finally {
        System.out.println("finally");
    }
}
```

<font color=red>★⑸</font>进入这个方法的线程条件是不是第一个线程节点或者是第一个线程节点，但争抢锁失败的。但是进入这个方法为什么只要当前线程的线程节点和他的前节点呢，其实整个队列都有可能需要修改，只是对于每一个线程来说只需要管理自己node和前节点的关系，从而影响到整个队列。AQS双向队列是多线程维护的数据结构，就除了head作为哨兵节点外，其他每一个线程节点对应的那个线程进行自旋。在这里每一个线程如果前驱节点的状态是线程取消状态的话，就需要过滤自己的前驱节点，向前迭代查找新的节点，所以利用每个线程寻找前驱节点来对那些线程取消的node进行过滤。但是貌似存在一个边界值的问题，如果tail元素是线程取消状态呢，tail元素可没有后继节点。但是问题不大，只需要其他线程获取锁失败进入AQS队列，成为新的tail，就能把前tail元素给过滤了，这里的边界值问题其实也就没有必要再过多的纠结了。但是这里的巧妙之处并不是只有这一点，从中也可以发现，后继元素的线程状态都是存储在前驱元素的状态中，也就是说第二个节点线程状态是存放在哨兵节点中，以此类推。总结：最终所有线程的node都会通过自旋shouldParkAfterFailedAcquire方法来过滤线程取消状态下的node，然后在通过CAS操作对前驱节点的waitStatus设置Node.SIGNAL，最终前驱节点waitStatus为true而调用LockSupport.park将线程挂起。<br />

<font color=red>ReentrantLock独占锁lock小总结1：</font>从ReentrantLock使用AQS来说，我现在将非公平锁的加锁过程讲解完毕，但是和我之前理解的非公平锁有点出入，我之前理解的非公平锁，多个线程操作同一个对象时，所有线程一块抢，所有线程包括进入队列中的。但是以目前来说，为什么只有队列里第一个线程node才有机会和外界整准备尝试争抢锁的线程去竞争，那队列后面的节点是没有机会去尝试获取锁的，这不是排队嘛，不成了公平锁了吗。再极端一点，如果现在很多线程已经进入队列，外界已经没有任何线程去争夺锁(锁是没有任何线程占有)，第一个线程node获取锁肯定能成功，转而退出AQS队列。这种排队等候的机制就很公平，总之呢，这里我是不能理解的，要想得到答案就需要将源码解析完，不然就是管中窥豹可见一斑。<br />

<font color=red>ReentrantLock独占锁lock小总结2：</font>其实应该线程争夺锁有三次机会，机会1：直接尝试去修改state状态，修改成功这自己占有。机会二：由于ReentrantLock是可重入锁，尽管在机会1中失败了，但是必要确认一下占锁的线程是否是自己，这段过程中还会去尝试去获取锁。机会三，：前两次都失败了，如果前驱元素是head的线程node的线程，还会尝试去获得锁。如果尝试失败或者是前驱不是head，前驱线程状态是SIGNAL，就需要将其挂起，这种挂起是要通过操作系统内核的，绝对是重量级的。之所以要这么使用，是为了方便AQS去适应灵活的并发场景。其实在一些低并发的情况下，AQS队列长度不是很长，甚至没有，这种情况的线程数量很少，CAS去获取锁的成功率很大，而且用完就下一个node去争抢，在这种线程少的情况下是没必要通过系统内核阻塞线程。线程少的情况下用CAS自旋的这种方式会更加的轻量级。但是一旦在高并发甚至是超高并发的场景下，AQS队列中的元素很多，这就代表着线程的数量就很多，如果大量的线程进行一个自旋，会严重占用CPU资源，而且线程上下文切换频繁，所带来的开销不如将这些线程直接阻塞挂起。这也算是很常见的一个结论，在低并发的场景下，更多的是选择CAS自旋这种轻量级的方式去解决，而在高并发场景下，肯定是选更加重量级的，当然也可以将这些理解为悲观锁和乐观锁的选择。<br />