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
        // 首先当前线程会先尝试获取锁，由于state默认是0，通过CAS操作来获取锁，一旦获取锁成功
        // 就保存当前线程为资源的占有线程
        // 由于是非公平锁，谁争抢到就是谁的
        if (compareAndSetState(0, 1))
            // setExclusiveOwnerThread是由AQS继承AbstractOwnableSynchronizer类而来
            // 里面有一个exclusiveOwnerThread属性，用来保存拥有者线程，并提供set-get方法。
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 如果CAS失败，说明线程占着锁，但是也有可能这个线程是自己吧
            // 调用AQS中的acquire方法，该方法在AQS中是模板方法，下面重写了
            acquire(1);
    }

    // 尝试获取独占锁
    protected final boolean tryAcquire(int acquires) {
        // 调用继承Sync的方法
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
        // 公平锁不会像非公平锁一上来去尝试去获取锁，为什么？
        // 因为当前线程是后来的，要先确保队列中的元素先去抢，这就是公平和非公平的区别
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        // 获取当前线程
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // hasQueuedPredecessors为公平策略，移步至AQS
            // 如果返回false，说明有权利争抢
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            // 进行重入
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            // 偏向，不需要CAS保证原子性
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
        // 获取当前线程
        final Thread current = Thread.currentThread();
        // 获取当前状态
        int c = getState();
        // 如果是0，说明资源被锁，nonfairTryAcquire方法就是因为资源被锁而进来
        // 为什么这里还需要再次判断？
        // 有可能在你调用getState之前占用线程释放了锁，所以在这里判断是有必要的
        if (c == 0) {
            // 这里CAS操作获取
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果占用线程是我当前线程，则允许访问，所以说ReentrantLock是一个可重入锁，重入一次state就+1
        else if (current == getExclusiveOwnerThread()) {
            // acquires是我们传的，并没有直接写+1。
            int nextc = c + acquires;
            // 肯定不能为负数吧
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            // 设置状态
            // 这里没有通过CAS来设置，是因为本身这个锁已经被当前线程获取，只有一个线程可以改，实现了偏向锁的功能
            setState(nextc);
            return true;
        }
        return false;
    }


    // 尝试释放锁
    protected final boolean tryRelease(int releases) {
        // 获取状态-1
        int c = getState() - releases;
        // 查看当前线程是否是占有线程
        if (Thread.currentThread() != getExclusiveOwnerThread())
            // 不是就抛异常，主要还是进行线程身份检查，防止锁并没有被lock，而用调unlock，或者是当前线程不是占有线程
            throw new IllegalMonitorStateException();
        // 如果是占有线程，free默认为false
        boolean free = false;
        // ReentrantLock是可重入锁，也有可能重入n次
        // 如果state为0，要释放资源了
        if (c == 0) {
            // 返回flag为true
            free = true;
            // 线程争抢锁是以state值为条件,只要在state设置之前，把占有线程清理，是可以的
            // 这里起了一个偏向锁的作用
            setExclusiveOwnerThread(null);
        }
        // 此时设置state，其实在state设置前的所有操作，包括设置state操作
        // 因为偏向锁的作用下，直接设置都是线程安全的。
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

2. 释放锁：unlock()
```java
public void unlock() {
    // AQS继承过来的方法
    sync.release(1);
}
```