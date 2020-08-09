# InheritableThreadLocal源码解析

### 建议：在理解了ThreadLocal基础上再来阅读该源码解析

## Thread类中和threadLocals是一样的，线程私有

```java
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

**源码很少，继承了ThreadLocal类并且重写了三个方法**
```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    /**
     * Computes the child's initial value for this inheritable thread-local
     * variable as a function of the parent's value at the time the child
     * thread is created.  This method is called from within the parent
     * thread before the child is started.
     * <p>
     * This method merely returns its input argument, and should be overridden
     * if a different behavior is desired.
     *
     * @param parentValue the parent thread's value
     * @return the child thread's initial value
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
     * 获取当前线程的inheritableThreadLocals,把threadLocals覆盖
     *
     * @param t the current thread
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * inheritableThreadLocals代替了threadLocals
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the table.
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}

```

**InheritableThreadLocal主要是针对子线程可以获取到父线程中的Entry数据，相比较ThreadLocal就不能把父线程存放的数据复制到子线程。首先，什么叫父子线程，请看下面例子：**
```java
/**
 * 父子线程
 */
public class Demo1 {


    /**
     * 可以看得出来整个代码很少，但是完全可以说明问题，child线程是一个子线程，
     * 要想找到它的父线程，就得看是谁new该线程，很显然，这里是main线程
     * @param args
     */
    public static void main(String[] args) {
        Thread child = new Thread(() -> {
            System.out.println("我是子线程");
        });

        child.start();
    }
}
```

**为什么谁把子线程new了就是父线程，当然这种问题在伦理上是符合逻辑的，但是放在程序上就需要看源代码了。**

```java
// 跟踪Thread的构造函数
public Thread(Runnable target) {
    // 初始化方法
    // 1.ThreadGroup，我们是单个线程，肯定为null
    // 2.Runnable，需要执行的任务, 我们传进来的
    // 3.线程名字，固定模板加一个递增加锁的静态变量
    init(null, target, "Thread-" + nextThreadNum(), 0);
}

private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
    init(g, target, name, stackSize, null);
}

private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }

    this.name = name.toCharArray();
    // 获取当前new该线程的线程作为该线程的父线程
    Thread parent = currentThread();
    SecurityManager security = System.getSecurityManager();
    if (g == null) {
        /* Determine if it's an applet or not */

        /* If there is a security manager, ask the security manager
            what to do. */
        if (security != null) {
            g = security.getThreadGroup();
        }

        /* If the security doesn't have a strong opinion of the matter
            use the parent thread group. */
        if (g == null) {
            g = parent.getThreadGroup();
        }
    }

    /* checkAccess regardless of whether or not threadgroup is
        explicitly passed in. */
    g.checkAccess();

    /*
        * Do we have the required permissions?
        */
    if (security != null) {
        if (isCCLOverridden(getClass())) {
            security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
        }
    }

    g.addUnstarted();

    this.group = g;
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();
    if (security == null || isCCLOverridden(parent.getClass()))
        this.contextClassLoader = parent.getContextClassLoader();
    else
        this.contextClassLoader = parent.contextClassLoader;
    this.inheritedAccessControlContext =
            acc != null ? acc : AccessController.getContext();
    this.target = target;
    setPriority(priority);
    // 如果父线程有值
    if (parent.inheritableThreadLocals != null)
        // 复制一份给子线程，所以子线程就拿到了父线程存的值
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    /* Stash the specified stack size in case the VM cares */
    this.stackSize = stackSize;

    /* Set thread ID */
    tid = nextThreadID();
}

static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    // 给子线程new一个ThreadLocalMap，并将父线程丢进去初始化
    return new ThreadLocalMap(parentMap);
}

private ThreadLocalMap(ThreadLocalMap parentMap) {
    // 获取父线程的ThreadLocalMap中的键值对
    Entry[] parentTable = parentMap.table;
    // 获取长度
    int len = parentTable.length;
    // 设置子线程的阈值len * 2 / 3
    setThreshold(len);
    // 子线程的inheritableThreadLocals中ThreadLocalMap的Entry数组长度
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        // e为null的不处理
        if (e != null) {
            // 获取以ThreadLocal的key
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            // key不为null说明没内存泄露，父线程内存泄露的对象我们肯定不能要,而且key虽然是ThreadLocal引用，但是对象是InheritableThreadLocal
            if (key != null) {
                // 这里被重写了
                // 再通过InheritableThreadLocal重写的方法过渡一下
                Object value = key.childValue(e.value);
                // 重新new新的键值对
                Entry c = new Entry(key, value);
                // 计算索引值
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    // 直到找到为止
                    h = nextIndex(h, len);
                // 找到之后放入键值对
                table[h] = c;
                // 元素个数++
                size++;
            }
        }
    }
}
```

**下面我们通过ThreadLocal和InheritableThreadLocal做一个代码对比**

1. ThreadLocal

```java
/**
 * 父子线程
 */
public class Demo1 {

    private static ThreadLocal<String> local = new ThreadLocal<String>();

    public static void main(String[] args) {
        local.set("main线程存放");
        Thread child = new Thread(() -> {
            System.out.println("子线程读取：" + local.get());
        }, "子线程");
        child.start();
        System.out.println("main线程读取：" + local.get());
    }
}
```
**控制台打印出，可以看见子线程是不能获取父线程的存储值**
```console
main线程读取：main线程存放
子线程读取：null
```

2. InheritableThreadLocal
```java
/**
 * 父子线程
 */
public class Demo1 {

//    private static ThreadLocal<String> local = new ThreadLocal<String>();

    private static ThreadLocal<String> local = new InheritableThreadLocal<String>();

    public static void main(String[] args) {
        local.set("main线程存放");
        Thread child = new Thread(() -> {
            System.out.println("子线程读取：" + local.get());
        }, "子线程");
        child.start();
        System.out.println("main线程读取：" + local.get());
    }
}
```
**控制台打印出，可以看见子线程能获取父线程的存储值**
```console
main线程读取：main线程存放
子线程读取：main线程存放
```