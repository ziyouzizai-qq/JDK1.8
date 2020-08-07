# AbstractQueuedSynchronizer(简称AQS)源码解析

### 简介：AbstractQueuedSynchronizer抽象同步队列简称AQS，juc包下中锁的底层实现都是使用的AQS。这次源码解析我不会直接对AbstractQueuedSynchronizer进行单独解析，我会结合ReentrantLock一起解析，所以说这次还需要建立 ReentrantLock.md，一起来理解什么是AQS？

<hr />
## AQS的内部类Node

**Node**

```java
static final class Node {
    // 下面两个为一类标记
    // 用来标记该线程是获取共享资源时倍阻塞挂起后进入AQS队列，就是个标记
    static final AbstractQueuedSynchronizer.Node SHARED = new AbstractQueuedSynchronizer.Node();
    // 和上面SHARED一样起标记作用，目的是获取独占资源时阻塞挂起至AQS队列
    static final AbstractQueuedSynchronizer.Node EXCLUSIVE = null;

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
    volatile AbstractQueuedSynchronizer.Node prev;
    // 后节点
    volatile AbstractQueuedSynchronizer.Node next;
    // 存放进入AQS队列里面的线程
    volatile Thread thread;
    // 
    AbstractQueuedSynchronizer.Node nextWaiter;

    final boolean isShared() {
        return this.nextWaiter == SHARED;
    }

    final AbstractQueuedSynchronizer.Node predecessor() throws NullPointerException {
        AbstractQueuedSynchronizer.Node var1 = this.prev;
        if (var1 == null) {
            throw new NullPointerException();
        } else {
            return var1;
        }
    }

    Node() {
    }

    Node(Thread var1, AbstractQueuedSynchronizer.Node var2) {
        this.nextWaiter = var2;
        this.thread = var1;
    }

    Node(Thread var1, int var2) {
        this.waitStatus = var2;
        this.thread = var1;
    }
}
```

#### AQS的实例方法(独占锁获取和释放)，下面是独占方式下获取资源和释放资源时使用的方法，可以和ReentrantLock联系在一起，里面用了大量的接口调用和模板方法设计模式。
```java
// 独占锁获取
public final void acquire(int arg) {
    // tryAcquire其实在AQS中是一个模板方法形式存在，下面的tryRelease方法也是，很显然，这些都需要子类实现。
    // 以ReentrantLock为例来获取锁，请移步至ReentrantLock.md文档中ReentrantLock的实例方法lock()
    if (!tryAcquire(arg) &&
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