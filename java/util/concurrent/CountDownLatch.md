# CountDownLatch源码解析

## 1.简介

应用场景：需要主线程开启多个线程去并行执行任务，并且主线程需要等待所有线程执行完毕后进行汇总的场景。一般我们会使用线程的join，但是join却不够灵活，不能够满足不同场景，这个时候会使用CountDownLatch，使得代码会更加的优雅。CountDownLatch底层也是实现了AQS，有了前期的AQS为基础，CountDownLatch会更加容易理解他的工作原理。

## 2.CountDownLatch数据结构和边缘方法
```java
    // AQS实现类的实例
    private final Sync sync;

    // 传入计数的初始基数
    public CountDownLatch(int count) {
        // check
        if (count < 0) throw new IllegalArgumentException("count < 0");
        // 实例化AQS实现类的实例，并传入基数。
        this.sync = new Sync(count);
    }

	// 获取递减的剩余次数
    public long getCount() {
        return sync.getCount();
    }
```


## 3.Sync静态内部类
```java
// 与ReentrantLock相似，都是Sync继承AQS
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    // 构造器
    Sync(int count) {
        // 对AQS中的state设置
        // 次数state作为递减的次数
        setState(count);
    }

    // 获取当前还剩余的次数
    int getCount() {
        return getState();
    }

    // acquires其实是没有用到的
    protected int tryAcquireShared(int acquires) {
        // 判断当前值是否为0
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        // CAS自旋
        for (;;) {
            // 获取当前状态作为CAS的条件
            int c = getState();
            // 如果state为0，说明其他线程已经将基数减至0
            if (c == 0)
                // 结束自旋并返回false
                return false;
            // -1
            int nextc = c-1;
            // CAS设置state
            if (compareAndSetState(c, nextc))
                // CAS设置成功后，设置后state是否为0
                // 为0情况，返回true
                // 释放共享对象，其实只有将基数减至0的线程会触发释放机制，因为最后那条线程nextc == 0为true
                return nextc == 0;
        }
    }
}
```

## 4.CountDownLatch的核心方法

### 4-1 await()方法

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

// AQS
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 检查当前线程是否中断，则清除中断标志，并抛出中断异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 尝试获取锁，CountDownLatch的state是共享的
    // state不为0则说明递减没有结束，还需要等待，那就给我进入AQS队列吧
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

// Sync
protected int tryAcquireShared(int acquires) {
    // 查看state是否为0， 为0说明递减结束
    return (getState() == 0) ? 1 : -1;
}

// 这边代码参考ReentrantReadWriteLock中读锁
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```

### 4-2 countDown()方法

```java
public void countDown() {
    // 递减state释放共享锁
    sync.releaseShared(1);
}

// AQS
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        // 递减为0的线程来唤醒AQS队列
        doReleaseShared();
        return true;
    }
    return false;
}

// Sync
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    // CAS自旋
    for (;;) {
        // 获取state
        int c = getState();
        // 如果state为0，说明已经有其他线程递减完毕，所以返回false
        if (c == 0)
            return false;
        // 递减
        int nextc = c - 1;
        // CAS更新
        if (compareAndSetState(c, nextc))
            // CAS成功，查看是否是最后一次递减，递减成0的这条线程需要唤醒AQS队列中的节点
            return nextc == 0;
    }
}

private void doReleaseShared() {
    // 自旋
    for (;;) {
        // 每次自旋获取头节点
        Node h = head;
        // 如果头节点不为空，并且和尾节点不一致，一致说明只有一个哨兵节点或者tail为空，tail为空说明有线程已经在初始化AQS队列了
        // 只不过对头节点CAS成功后，还没有设置tail = head，就线程上下文切换了，因此产生head有值，tail为null的快照
        if (h != null && h != tail) {
            // 获取头节点的等待状态
            int ws = h.waitStatus;
            // 如果状态是SIGNAL，说明next节点被挂起了，则需要唤醒next节点中的线程，让这个线程继续执行
            if (ws == Node.SIGNAL) {
                // 通过CAS将SIGNAL设置成无状态，这边状态可以参考读写锁那边，有详细说明
                // doReleaseShared其实是一个共通方法，但是可以解决这里共享锁的释放
                // 为什么呢？因为能调用doReleaseShared方法对于CountDownLatch来说只有一个线程，就是最后一个将state递减为0的线程
                // 因此这里只有一个线程操作，第一次CAS肯定百分百成功，因此continue是不会走到的
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 唤醒节点，详细参考读写锁
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                // 将无状态改为释放共享资源时需要通知其他节点状态
                // CAS失败则自旋
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            // 如果头改变则退出自旋，说明AQS队列已经被激活了，没有必要唤醒了
            break;
    }
}
```



