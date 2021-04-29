# ReentrantReadWriteLock（jdk11）

## 1.简介

​		可重入的读写锁，写锁为独占锁，读锁为共享锁，大多适用于读多写少的场景。因为现在的版本解析的是11，11的版本来解析是因为这篇文章是我在公司写的，不过和8会存在出入，比如AQS中用VarHandle代替了unsafe，假设你会了AQS，不会也不要紧，我这边打算重新讲AQS，unsafe不了解的同学可以看AQS那一章，该篇我也会讲解VarHandle

## 2.数据结构

```java
// 读写锁，可见读写锁都是ReentrantReadWriteLock的内部类
private final ReentrantReadWriteLock.ReadLock readerLock;
private final ReentrantReadWriteLock.WriteLock writerLock;
// ReentrantReadWriteLock的内部类，继承了AQS
final Sync sync;
```

## 3.构造器

### 3-1.ReentrantReadWriteLock构造器

```java
// 大体思路和ReentrantLock一致
public ReentrantReadWriteLock() {
    // 默认是非公平锁
    this(false);
}

public ReentrantReadWriteLock(boolean fair) {
    // 可以选择公平锁和非公平锁
    sync = fair ? new FairSync() : new NonfairSync();
    // 将ReentrantReadWriteLock对象传入读写锁中，主要作用是获取sync，详情看第2点
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}
```

### 3-2.ReentrantReadWriteLock内部类

#### 3-2-1.ReadLock

```java
public static class ReadLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -5992448646407690164L;
        private final Sync sync;
    	// 从ReentrantReadWriteLock对象中获取sync并保存
    	protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
    
    	// ...
}
```

#### 3-2-2.WriteLock

```java
public static class WriteLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -4992448646407690164L;
        private final Sync sync;
    	// 从ReentrantReadWriteLock对象中获取sync并保存
    	protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
    
    	// ...
}
```

#### 3-2-3.Sync及其子类（FairSync&NonfairSync）

```java
// 公平锁
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    // 公平锁需要进行公平策略
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}

// 非公平锁
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    // 非公平固定为false，目的是直接CAS
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    final boolean readerShouldBlock() {
        /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
        return apparentlyFirstQueuedIsExclusive();
    }
}
```

## 4.设计原理及源码解析

从上面的代码结构来看，原理还需要看Sync和AQS，建议将源码打开结合注释观看

### 4-1.WriteLock的lock

> 当我们获取写锁并操作写锁时，由于写锁是一个排他锁，只能一个线程操作，具体看源码，假设以**非公平锁**作为解析对象
>

```java
// sync就是WriteLock中保存的，此处为非公平锁，我们以非公平锁作为解析对象的嘛
public void lock() {
    // 该方法就是我们AQS中的方法了，tryAcquire用的模板方法，具体看继承AQS的Sync
    // 传入1，用于加锁次数
    sync.acquire(1);
}

// AQS与1.8略有出入，重新在读一遍源码
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 当前线程设置中断标志
        selfInterrupt();
}

static void selfInterrupt() {
    // 当前线程设置中断标志
    Thread.currentThread().interrupt();
}

// 获取锁失败，进入AQS队列
private Node addWaiter(Node mode) {
    // 查看Node构造器
    Node node = new Node(mode);

    for (;;) {
        // 获取尾结点
        Node oldTail = tail;
        // 不为null
        if (oldTail != null) {
            // 设置当前线程节点的前节点为尾节点
            node.setPrevRelaxed(oldTail);
            // 试试能否通过CAS将尾节点改为当前线程节点
            if (compareAndSetTail(oldTail, node)) {
                // CAS成功，则将旧尾节点的next节点设置成当前节点，因此AQS是双向的，这是不存在线程问题的，因为CAS是硬件级别的原子性操作
                // CAS失败，则重新循环，再获取新的尾节点，因此进入AQS队列采取的是CAS自旋的方式。
                oldTail.next = node;
                // 返回节点
                return node;
            }
        } else {
            // 尾节点还未存在，说明还没有线程操作，则初始化AQS队列
            initializeSyncQueue();
        }
    }
}

// 初始化AQS双向队列
private final void initializeSyncQueue() {
    Node h;
    // 了解AQS双向队列的朋友会知道，头节点其实是一个无线程节点，又称之为哨兵节点
    // 头节点的设置也是通过CAS来确保线程安全的
    // 如果CAS失败，说明恰好有线程在此时初始化了头节点，导致头节点有值，而不是null
    if (HEAD.compareAndSet(this, null, (h = new Node())))
        // 如果CAS成功，说明初始化AQS队列的是当前线程
        // 则头尾节点一致
        // 这边存在一个细节，由于是通过CAS头节点来判断是否成功初始化AQS队列
        // CAS头节点存在原子性，但是并不是设置头节点和设置尾节点存在原子性
        // 因此存在CAS成功头节点有值，而尾节点为null的情况，因为线程存在上下文切换
        // 因此在后面的处理中一定要注意这边的问题
        tail = h;
}


final boolean acquireQueued(final Node node, int arg) {
    // 设置当前线程中断的flag，false表示不中断
    boolean interrupted = false;
    try {
        for (;;) {
            // 获取当前线程节点的前节点
            final Node p = node.predecessor();
            // 如果前节点是头节点，说明当前线程节点为第一个线程节点，因为头节点为哨兵节点嘛
            // 如果是，则去尝试获取锁，因此，只有AQS中第一个线程节点才是有机会和外界线程去争取独占锁的
            if (p == head && tryAcquire(arg)) {
                // tryAcquire我们解析过了，每一个AQS实现不同
                // 如果获取锁成功，则设置头节点
                setHead(node);
                // 将旧的头节点的next设置为null，方便GC回收掉
                p.next = null; // help GC
                // 返回中断标志
                return interrupted;
            }
            // 那如果当前节点不是第一个线程节点或者是第一个线程节点争抢锁失败呢
            if (shouldParkAfterFailedAcquire(p, node))
                // 如果node节点的状态已经设置到前节点上了，我们就可以大胆放心的将这个线程阻塞挂起
                // 如果该线程被中断，则返回true并且清理中断标记，出去还是被中断，哈哈
                // 没有被中断过就看interrupted
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        if (interrupted)
            selfInterrupt();
        throw t;
    }
}

/**
* volatile int waitStatus:
*      0: 无状态(默认就是0嘛)
*      CANCELLED(1)：线程被取消了
*      SIGNAL(-1): 线程需要被唤醒
*      CONDITION(-2): 条件队列的中节点的标记
*      PROPAGATE(-3): 释放共享资源时需要通知其他节点
*
**/
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获取前节点的等待状态，如果next的节点需要被唤醒，则将前节点状态变为SIGNAL
    int ws = pred.waitStatus;
    // 假设节点进入AQS队列，不是第一个线程节点或者是第一个线程节点争抢锁失败，第一次进入shouldParkAfterFailedAcquire
    // 此时ws上为0，则需要将可被唤醒的状态设置到前节点上
    if (ws == Node.SIGNAL)
        // 如果当前node的状态已经设置到前节点上并且是可被唤醒的，我们就可以安全的将线程挂起了。
        return true;
    if (ws > 0) {
        // ws大于0，说明pred节点被取消了，就需要过滤
        // 这边有一个细节，如果说pred为head，node为第一个线程节点，争抢锁失败嘛
        // pred = pred.prev就是null，pred.waitStatus会不会空指针啊
        // 答案是不会，头节点是哨兵节点，不是线程节点，因此不存在线程取消状态嘛
        // 那又有人问了，头节点除了AQS队列初始化生成的，不是还有第一个线程节点变成head的嘛，你看setHead也并没有将waitStatus设置成0啊
        // 会不会有遗留的状态呢，这个问题也是我中途解析的时候发现的，好像目前来说没办法解释
        // 但是可以肯定的是在head情况下100%不存在CANCELLED，在其他地方肯定是有处理。我们带着这个问题遇到再说
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        // 过滤掉CANCELLED节点
        pred.next = node;
    } else {
		
        // 将前节点的状态设置成可被唤醒的，保证后面节点被唤醒
        // 因此设置之后还有一次机会出去自旋一次，自旋无果走上面ws == Node.SIGNAL分支返回true，就被park
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}

private void setHead(Node node) {
    // 将当前节点设置成头节点，由于获得了锁了，当前线程节点肯定会退出AQS队列（将头节点踢掉，让自己成为头节点）
    // 设置头节点为当前node
    head = node;
    // 头节点特征是无线程状态，则要设置为null
    node.thread = null;
    // 头节点没有前驱节点嘛，则要设置为null
    node.prev = null;
}

Node(Node nextWaiter) {
    // 将该节点的节点状态设置成独占，表示独占节点
    this.nextWaiter = nextWaiter;
    // 将Node线程设置成当前线程
    THREAD.set(this, Thread.currentThread());
}

// AQS中的非公平策略
public final boolean hasQueuedPredecessors() {
    Node h, s;
    if ((h = head) != null) {
        if ((s = h.next) == null || s.waitStatus > 0) {
            s = null; // traverse in case of concurrent cancellation
            for (Node p = tail; p != h && p != null; p = p.prev) {
                if (p.waitStatus <= 0)
                    s = p;
            }
        }
        if (s != null && s.thread != Thread.currentThread())
            return true;
    }
    return false;
}

// Sync： AQS的模板方法，尝试成功就返回true，失败就返回false，并进入AQS队列
@ReservedStackAccess
protected final boolean tryAcquire(int acquires) {
    // 获取当前线程
    Thread current = Thread.currentThread();
    // 获取AQS的状态值，这个状态值对于每个AQS的实现来说是不一样的，见脚注【1】
    int c = getState();
    // 将状态值通过exclusiveCount方法获取当前的写锁的重入次数
    int w = exclusiveCount(c);
    // 如果state状态不为0，可以得出当前写锁已经其他线程获取的结论吗？
    // 答案是不可以，因为state存放的是两种状态，有可能c!=0是由读锁被获取造成的
    if (c != 0) {
        // w为0说明当前有其他线程在读，如果此时获取写线程，容易导致别的读线程脏读，不可重复读，幻读，因此尝试获取失败
        // w不为0说明没有线程在读，而c不为0，说明有写线程在操作，判断持有锁线程是否为当前线程，不是则尝试获取失败
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 运行到这说明state只有写状态，并且持有锁的线程是当前线程
        // 就需要判断是否到达最大可重复次数
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        // 设置state，读锁低16位，直接加就OK了
        setState(c + acquires);
        // 获取成功
        return true;
    }
    // c== 0的情况，说明该读写锁第一次操作
    // 非公平锁返回false
    // 返回false的目的是直接CAS，因为是或运算嘛
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        // CAS失败说明被其他线程操作，有可能在读有可能在写，总之state变了
        return false;
    // 运行至此说明CAS成功，设置持有线程为当前线程，并返回true，尝试成功
    setExclusiveOwnerThread(current);
    return true;
}
```

```java
// Sync类
abstract static class Sync extends AbstractQueuedSynchronizer {
    static final int SHARED_SHIFT   = 16;
    // 共享锁(读锁的)的状态值
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    // 共享锁(读锁的)的最大个数
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    // 独占锁(写锁)的掩码，二进制15个1
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
    
    /** 返回共享的次数，就是读锁的获取次数，由于高16位存放的是读锁的次数，所以说要无符合右移16位才能得到 */
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    /** 返回写锁的可重入次数，由于低16位存的是写锁的可重入次数，做&运算可得，因此写锁的可重入次数最大值就是EXCLUSIVE_MASK */
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
}
```



[^1]: 对于读写锁来说，`state`值，整个`state`值是int类型，所以为32位，而读写锁就将这32位分为高16位和低16位两种状态来处理，因此`state`的十进制并不像ReentrantLock那样有实际的意义，这种将几种状态存放到一个值中的手法很常见，例如：线程池等。

### 4-2.WriteLock的unlock

上面讲了WriteLock的lock操作

```java
// 释放锁
public void unlock() {
    sync.release(1);
}

// AQS
public final boolean release(int arg) {
    // 尝试释放锁，和tryAcquire一样，AQS模板方法，具体看Sync
    if (tryRelease(arg)) {
        // 如果一旦释放锁，之前在AQS队列里面有可能会存在线程节点被挂起
        // 就需要释放锁的线程去将这个节点唤醒
        // 获取头节点
        Node h = head;
        // 看看头节点的等待状态
        if (h != null && h.waitStatus != 0)
            // 处理状态有状态的头节点，0位无状态嘛，有可能此时第一个线程节点还是无状态的，还在和外界线程竞争锁(非公平锁需要竞争)
            // 有可能争抢失败，将头节点状态改为SIGNAL，此时释放锁则进入unparkSuccessor
            unparkSuccessor(h);
        return true;
    }
    return false;
}

/*
*      0: 无状态(默认就是0嘛)
*      CANCELLED(1)：线程被取消了
*      SIGNAL(-1): 线程需要被唤醒
*      CONDITION(-2): 条件队列的中节点的标记
*      PROPAGATE(-3): 释放共享资源时需要通知其他节点
*/
private void unparkSuccessor(Node node) {
    // head节点进入，获取waitStatus
    int ws = node.waitStatus;
    // 如果说小于0，三种状态：SIGNAL、(CONDITION、PROPAGATE),括号这两个状态其实是不存在的
    // CONDITION用在条件队列，当不满足某种条件时，会存在条件队列，里面节点也是Node，不过标记成这个状态
    // 因此在AQS队列中不会存在条件Node，当然，条件队列里面的节点当满足条件会放入AQS队列，状态会进行"洗白"
    if (ws < 0)
        // 将状态改为无状态
        node.compareAndSetWaitStatus(ws, 0);

	// 获取next节点
    Node s = node.next;
    
    if (s == null || s.waitStatus > 0) {
        // 如果没有next节点，或者next节点的状态被取消了
        s = null;
        // 从尾节点往前找，将AQS队列中的CANCELLED节点过滤掉
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)
                s = p;
    }
    if (s != null)
        // 如果存在next节点，又因为node状态为SIGNAL，则s节点需要唤醒
        // 因为s有可能被过滤导致其waitStatus并不是SIGNAL，而是0这种无状态，说明线程并没有被阻塞挂起，LockSupport.unpark调用也没事
        LockSupport.unpark(s.thread);
}

// Sync
protected final boolean tryRelease(int releases) {
    // 如果当前线程并不持有锁，则抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 获取写锁的重入次数，由于是低16位，直接减次数就ok了
    int nextc = getState() - releases;
    // 如果递减后的可重入次数为0
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        // 则将持有线程设置为空
        setExclusiveOwnerThread(null);
    // 设置状态，见脚注【2】
    setState(nextc);
    // 如果释放成功，则返回true，释放不成功则说明可重入递减吧
    return free;
}

protected final boolean isHeldExclusively() {
    // 查看当前持有锁的线程是否为当前线程
    return getExclusiveOwnerThread() == Thread.currentThread();
}
```

[^2]: 这里一定要注意设置顺序，不可以先设置状态再设置持有线程为空，因为lock时候都是以state为加锁标准，如果在unlock时候不保证所有操作在设置state之前完成，会存在线程安全问题。例如两个锁的场景，锁A和锁B，先锁A后锁B，解锁的时候就得先解锁B，再解锁A，又有点像栈，“先进后出”。

### 4-3.WriteLock总结

其实WriteLock是比较容易理解的，和ReentrantLock非常的相似，如果这块能明白，大家可以尝试自己阅读ReentrantLock源码。

### 4-4.ReadLock的lock

写锁是一个独占锁，而读锁就是一个共享锁，因此，ReentrantReadWriteLock将AQS的两块锁都涉及到，不得不佩服Doug Lea多线程功力之强。

```java
public void lock() {
    sync.acquireShared(1);
}

// AQS 
public final void acquireShared(int arg) {
    // 尝试获取共享锁
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

// AQS，和acquireQueued类似
private void doAcquireShared(int arg) {
    // 添加到AQS队列中，设置为共享节点，因此，AQS队列中独占，共享节点都存在
    final Node node = addWaiter(Node.SHARED);
    boolean interrupted = false;
    try {
        for (;;) {
            final Node p = node.predecessor();
            // 第一个线程节点争抢锁
            if (p == head) {
                // 尝试争抢锁，成功就返回true
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            // 挂起
            if (shouldParkAfterFailedAcquire(p, node))
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    } finally {
        if (interrupted)
            selfInterrupt();
    }
}

@ReservedStackAccess
protected final int tryAcquireShared(int unused) {
	// 获取当前线程
    Thread current = Thread.currentThread();
    // 获取state
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        // 如果当前写锁被获取了并且持有写锁的线程并非当期线程，则返回-1判定
        return -1;
    // 至此，说明写锁没有被获取，则获取共享的次数
    int r = sharedCount(c);
    // 非公平锁重写
    // 条件1：AQS队列中:第一个线程节点为共享节点
    if (!readerShouldBlock() &&
        // 条件2：共享的次数小于最大值
        r < MAX_COUNT &&
        // 条件3：将state高16位+1，但是这边由于是高16位，不是+1，而是+SHARED_UNIT，因此tryAcquireShared方法的参数名叫unused，没有被使用。
        compareAndSetState(c, c + SHARED_UNIT)) {
        // CAS成功说明获取到读锁
        // 获取读锁成功后，接下来是把一些数据改一改
        if (r == 0) {
            // 共享次数为0说明第一次读
            // 脚注【5】
            // 在Sync类中保存第一个读的线程
            firstReader = current;
            // 在Sync类中保存第一个读的线程当前重入次数
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            // 如果读线程为当前线程，重入次数递增
            firstReaderHoldCount++;
        } else {
            // 非第一个读的线程的情况
            // 见脚注【3】，数据结构见HoldCounter
            // cachedHoldCounter为最后一个读线程及其获取读锁的可重入次数
            HoldCounter rh = cachedHoldCounter;
            // 如果还未有最后一个读线程
            if (rh == null ||
                // 当前线程也并非为最后的读线程
                rh.tid != LockSupport.getThreadId(current))
                // 更新最后一次读线程为当前线程，见脚注【4】
                // readHolds详细讲移步到下一个代码块
                // 如果说当前线程中没有通过readHolds做操作，获取当前线程的初始化HoldCounter信息
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                // cachedHoldCounter不为null，并且最后一个读线程就是当前线程的情况
                // 但是还要cachedHoldCounter可重入次数为0
                // 这种情况就是当前线程是最后一个读线程，并且前一次释放过锁，现在又进来读了
                // 因此在将cachedHoldCounter再回写进当前线程中，防止线程清除过threadLocals容器
                readHolds.set(rh);
            // 可重入次数+1
            rh.count++;
        }
        return 1;
    }
    // 以上三个条件不满足，则进入
    // AQS为第一个线程节点为争抢写锁的线程节点(因为存在写操作，所以就放弃这次读操作了，会不会有人说以此类推后面都是写线程节点，MD！都说是读多写少)
    // 共享次数上限
    // CAS失败(说明此时有其他线程在操作，有可能是写线程，有可能是读线程)
    // 再尝试一下
    return fullTryAcquireShared(current);
}

final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    // 自旋
    for (;;) {
        // 获取读写锁state
        int c = getState();
        // 如果当前写锁被获取了并且持有写锁的线程并非当期线程，则退出自旋，返回-1判定进入队列
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) { // AQS队列中:第一个线程节点为独占节点，但是还没写，因为exclusiveCount(c)=0
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                // 当前资源还没有写过，但是写操作还在AQS队列
                // 确保第一个读线程可重入次数大于0，因此这种情况是允许他继续读的
            } else {
                // 如果当前线程不是第一个读线程
                if (rh == null) {
                    // 第一次自旋，获取最后一个读线程
                    rh = cachedHoldCounter;
                    // 没有最后一个读线程或者是当前线程不是最后一个读线程
                    if (rh == null ||
                        rh.tid != LockSupport.getThreadId(current)) {
                        // 获取当前线程的HoldCounter，没有就是初始值
                        rh = readHolds.get();
                        // 如果是初始值为0， 移除掉
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                // 如果发现之前都没重入过，就去排队
                // 其实rh的值只有以上两种情况
                // 第一种：记录的最后读线程
                // 第二种：通过ThreadLocal从线程中去出来
                // 因此如果是第一种情况，说明该线程是最后一个读线程，不过重入次数为0，说明之前释放过锁，现在又开始读，而且AQS队列中有写操作
                // 读的过于频繁，给我进队列，让人家先写
                // 第二种就是，如果是0说明这个线程是第一次读，之前都没读过，后面就有写操作，你就别忙着读，因此返回-1给我进队列去，而且
                // 又将存的值remove掉(readHolds.remove())
                // 这边主要还是处理非第一个读线程的情况，这些读线程如果都没有重入次数，就给我进队列等待，其他的情况就可以继续执行CAS争抢锁了
                if (rh.count == 0)
                    return -1;
            }
        }
        // 读取次数超过阈值，抛异常
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        
        // CAS获取共享锁
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            // 其实这块代码和tryAcquireShared中CAS成功一样，就不重复读了
            // 还没有读线程
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null ||
                    rh.tid != LockSupport.getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            // 退出自旋返回判定
            return 1;
        }
    }
}

/**
* Nonfair version of Sync
*/
final boolean readerShouldBlock() {
    return apparentlyFirstQueuedIsExclusive();
}

// AQS 检查AQS队列中第一个线程节点是否为共享节点
// 独占节点为true
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}

static final class HoldCounter {
    // 可重入次数
    int count;          // initially 0
    // Use id, not reference, to avoid garbage retention
    // 不保存线程对象，防止线程对象无法回收，因此获取的是线程ID
    final long tid = LockSupport.getThreadId(Thread.currentThread());
}

```

[^3]: cachedHoldCounter用来记录最后一个获取读锁的线程及其可重入次数
[^4]: ThreadLocal，这边还涉及到ThreadlLocal原理，假设我认为大家ThreadLcoal会使用，他的代码我也解析过了，可以去了解了解

```java
// ThreadLcoal篇，这边是对readHolds这块详细说明，首先该变量是Sync类

abstract static class Sync extends AbstractQueuedSynchronizer {
    
    private transient ThreadLocalHoldCounter readHolds;
    
    Sync() {
        // 首先当我们new Sync子类时，即公平锁和非公平锁，就已经初始化完成
        readHolds = new ThreadLocalHoldCounter();
        setState(getState()); // ensures visibility of readHolds
    }
    
    // ThreadLocalHoldCounter继承ThreadLocal，并重写了initialValue方法
    // 如果当前线程都没有通过readHolds去存东西，则第一次调用get方法时候就是initialValue返回值
    // 而new HoldCounter操作，触发count为0，这是零值，最重要的是获取到当前线程ID
    // 如果之前set过值，就直接获取set的值
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }
    
    static final class HoldCounter {
        int count;          // initially 0
        // Use id, not reference, to avoid garbage retention
        final long tid = LockSupport.getThreadId(Thread.currentThread());
    }
    
    // 脚注【5】
    private transient HoldCounter cachedHoldCounter;
    private transient Thread firstReader;
    private transient int firstReaderHoldCount;
    
}
```

[^5]: 这边读锁设置firstReader和firstReaderHoldCount,包括这个cachedHoldCounter，因为是读锁，是允许多线程同时操作的，不符合先行发生原则上任何一条规则，因此是存在线程安全问题的，那为什么对于firstReader和firstReaderHoldCount不做任何处理，主要是因为这两个变量不是特别重要，其次是因为这两个变量只在第一次读锁进来设值时产生线程安全，假设在第一时间点，很多读线程进行读，真的只有一个线程设的值让所有线程可见后才能成功，即使这个线程不是最先读的，只要先让所有线程可见。这样的话也可以认为他是第一个读的线程

### 4-4.ReadLock的unlock

上面看完了读锁的加锁，现在来解析读锁

```java
// ReadLock
public void unlock() {
    sync.releaseShared(1);
}

// AQS
public final boolean releaseShared(int arg) {
    // 尝试释放锁
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

// Sync
@ReservedStackAccess
protected final boolean tryReleaseShared(int unused) {
    // 获取当前线程
    Thread current = Thread.currentThread();
    // 如果当前线程是第一个读线程
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            // 如果重入次数为1， 将第一个读线程设置为null
            firstReader = null;
        else
            // 否则重入次数--
            firstReaderHoldCount--;
    } else {
        // 不是第一个读线程，则获取最后一个读线程
        HoldCounter rh = cachedHoldCounter;
        if (rh == null ||
            rh.tid != LockSupport.getThreadId(current))
            // 如果cachedHoldCounter为null，或者不是最后一个读线程
            // 获取当前线程的HoldCounter
            rh = readHolds.get();
        // 获取当前线程的重入次数
        int count = rh.count;
        if (count <= 1) {
            // count为1就移除掉，因为解锁了
            readHolds.remove();
            // 如果都没加锁过，你解啥锁？抛异常
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        // 重入次数递减
        --rh.count;
    }
    // 由于是共享锁，因此可以先将一些值先操作了，然后再CAS自旋
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            // 如果 state为0，则允许等待的写入程序继续。
            return nextc == 0;
    }
}


private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}


```



