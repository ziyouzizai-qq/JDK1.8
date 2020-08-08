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
        // 后去前节点，空指针检查
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

1. state：由于AQS我们都不会直接去使用，都是选用并发包下的锁，所以对于对于AQS的不同实现来说state所代表的意思是不一样的，就拿ReentrantLock来说，state表示当前线程获取锁的可重入次数。当然，对于像semaphore，ReentrantReadWriteLock等又是不同的意思。
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
            // 前驱不是head或者是争抢锁失败
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 利用LockSupport的park方法挂起并做中断判断
                parkAndCheckInterrupt())

                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 查看前驱节点的线程等待状态
    int ws = pred.waitStatus;
    // 如果pred是head，ws为默认值0
    // 前驱节点需要被唤醒的情况返回true
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    // CANCELLED(线程在队列里等待)的情况
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
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