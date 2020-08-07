# ReentrantLock源码解析

**ReentrantLock是一个轻量级的可重入锁，它的底层是用AQS实现的。ReentrantLock可以分为公平锁和非公平锁，默认是非公平锁，这次为了更好地去理解AQS，顺带把ReentrantLock源码解析了。首先还是的看类结构：**

#### ReentrantLock的数据结构
```java
// Sync其实我们不知道是什么类型，但是这行代码可以看出是常量，必须在构造器里注入，请看下面的构造器
private final Sync sync;
```


#### ReentrantLock的构造函数
```java
// 刚才我说过，ReentrantLock分为公平锁和非公平锁，无参默认就是非公平，其实从类名可以看得出来，先不纠结实现细节。
public ReentrantLock() {
    sync = new NonfairSync();
}

// 提供选择，true为公平锁，false为非公平锁，大家可以体会一下，定义了一个常量，而且必须在构造器里注入，还可供选择。
// 这种机制有没有单选框的味道。之所以默认是非公平锁，不是公平锁，是因为公平锁的并发效率太低，这里我先提一嘴，后面我会填坑。
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

#### sync类：AQS的子类，为ReentrantLock的静态抽象内部类，所以说ReentrantLock的底层就是用AQS去实现的。

#### sync类的两个实现类NonfairSync和FairSync

**NonfairSync**
```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        // 非公平锁如何实现lock
        // compareAndSetState是AQS继承过来的，移步至AbstractQueuedSynchronizer.md中unsafe类
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

**FairSync**
```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    // Sync中的抽象方法重写，直接调用AQS中获取独占锁
    final void lock() {
        // 然而此时已经对tryAcquire进行重写，好好感受JDK并发组中高手对于接口模板设计的技术手法。
        acquire(1);
    }

    /**
        * Fair version of tryAcquire.  Don't grant access unless
        * recursive call or no waiters or is first.
        */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

**Sync**
```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    /**
     * Performs {@link Lock#lock}. The main reason for subclassing
     * is to allow fast path for nonfair version.
     */
    abstract void lock();

    /**
     * Performs non-fair tryLock.  tryAcquire is implemented in
     * subclasses, but both need nonfair try for trylock method.
     */
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // Methods relayed from outer class

    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }

    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    final boolean isLocked() {
        return getState() != 0;
    }

    /**
     * Reconstitutes the instance from a stream (that is, deserializes it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```

#### ReentrantLock的实例方法

1. 获取锁：lock()
```java
// 接口调用实例方法，根据你注入的锁去调用，假设锁是公平锁
public void lock() {
    // 调用公平锁的lock方法
    sync.lock();
}
```