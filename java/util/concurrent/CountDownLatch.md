# CountDownLatch源码解析

#### 应用场景：需要主线程开启多个线程去并行执行任务，并且主线程需要等待所有线程执行完毕后进行汇总的场景。一般我们会使用线程的join，但是join却不够灵活，不能够满足不同场景，这个时候会使用CountDownLatch，使得代码会更加的优雅。CountDownLatch底层也是实现了AQS，有了前期的AQS为基础，CountDownLatch会更加容易理解他的工作原理。

## CountDownLatch的数据结构和方法
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

    // 当线程调用CountDownLatch对象await，当前线程会被阻塞。
    // ★⑵
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    // 释放共享锁，=> AbstractQueuedSynchronizer.md
    public void countDown() {
        sync.releaseShared(1);
    }

    public long getCount() {
        return sync.getCount();
    }
```


## CountDownLatch的Sync静态内部类
```java
// 与ReentrantLock相似，都是Sync继承AQS
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    // 构造器
    Sync(int count) {
        // 对AQS中的state设置
        // ★⑴详情
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
                return nextc == 0;
        }
    }
}
```

<font color=red>★⑴</font>在讲ReentrantLock时候，曾说AQS是JUC包下锁的基类，对于不同类型的锁来说，AQS中的state是含有不同的意义。像之前的ReentrantLock，state代表的是可重入的次数，而CountDownLatch，代表的就是计数器。<br />

<font color=red>★⑵</font>但是也不会一直阻塞。下面任一情况就会返回，情况一：当所有线程调用CountDownLatch对象的countDown方法后，一旦计数值为0时，返回。情况二：其他线程中断了当前线程，抛出InterruptedException异常，即返回。<br />