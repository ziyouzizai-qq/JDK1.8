# CopyOnWriteArrayList源码解析

**下面的一些对容器的增删改查的操作我都会挑出一个进行源码解析，因为原理一样，搞明白一个就行了。**

#### CopyOnWriteArrayList的数据结构
```java
// 锁
/** The lock protecting all mutators */
final transient ReentrantLock lock = new ReentrantLock();

// 底层数组
/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;
```

#### CopyOnWriteArrayList的构造器

1. 无参构造
```java
/**
 * Sets the array.
 */
final void setArray(Object[] a) {
    array = a;
}

/**
 * Creates an empty list.
 */
public CopyOnWriteArrayList() {
    // 初始化数组，容量0
    setArray(new Object[0]);
}
```

2. 有参构造
```java
// toCopyIn的副本作为数据源
public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}

// 旧容器
public CopyOnWriteArrayList(Collection<? extends E> c) {
    // 元素数组
    Object[] elements;
    // 如果是CopyOnWriteArrayList类型的对象，直接获取底层数组
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        // 接口调用，获取转换后的数组
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    // 设置当前CopyOnWriteArrayList对象的数组
    setArray(elements);
}
```

#### CopyOnWriteArrayList的实例方法
1. 添加元素：add(E e)
```java
public boolean add(E e) {
    // 获取独占锁
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取数组
        Object[] elements = getArray();
        // 获取数组长度
        int len = elements.length;
        // 扩容+1，newElements和elements不是同一个对象，装的是同一个对象
        // 但是会产生一个问题, 详情见★⑴
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 最后位+1
        newElements[len] = e;
        // 设置数组
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

2. 获取指定下标元素：get(int index)
```java
public E get(int index) {
    // 获取数组，并给予下标
    // 整个获取数组是没有通过获取锁进行
    return get(getArray(), index);
}

final Object[] getArray() {
    return array;
}

// 注意，不要下标越界，官方没有对下标进行检查
private E get(Object[] a, int index) {
    return (E) a[index];
}
```

3. 修改指定元素： set(int index, E element)
```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        // 获取指定下标元素
        E oldValue = get(elements, index);
        // 不一样就修改
        if (oldValue != element) {
            // 获取数组长度
            int len = elements.length;
            // copy一个新的数组快照
            Object[] newElements = Arrays.copyOf(elements, len);
            // 指定下标修改
            newElements[index] = element;
            // 设置数组
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            // 新旧值一样，为了保证volatile语义，重新设置
            setArray(elements);
        }
        // 返回旧值
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

4. 删除元素：remove(int index)
```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 获取旧值，用来返回
        E oldValue = get(elements, index);
        // 需要移动的数量
        int numMoved = len - index - 1;
        // 这种情况就是index为len-1，就是删除最后一位
        if (numMoved == 0)
            // 直接设置
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            // 分为两段复制
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                                numMoved);
            // 再设置
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

<font color=red>★⑴</font>由于获取元素没有通过加锁同步，而添加操作会有加锁操作，这就说明了，当其他线程在添加元素的过程中，我是可以获取元素的，而在添加元素过程中，由于Object[] newElements = Arrays.copyOf(elements, len + 1)，如果获取元素操作是在我扩容之前，获取元素而获取的数组很有可能是拿的旧的数组，其实这个时候其他线程又添加了新的元素，生成新的数组，所以，此时获取元素所获取的数组是之前数组版本的一个快照，而形成了一个写时复制的弱一致性问题。当然，不仅仅是和add，和其他修改操作都有这种问题。包括的迭代器也存在弱一致性问题。让我们一起用迭代器来感受感受什么是弱一致性问题


**获取迭代器**
```java
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);
}
```

```java
static final class COWIterator<E> implements ListIterator<E> {
        /** Snapshot of the array */
        private final Object[] snapshot;
        /** Index of element to be returned by subsequent call to next.  */
        private int cursor;

        private COWIterator(Object[] elements, int initialCursor) {
            // 指针，默认是0
            cursor = initialCursor;
            // 当获取迭代器时，会把数组给迭代器，但是一旦给了之后，想添加元素
            // 当前迭代器的数组就成了旧的数组，如果想获取新的数组，只能重新获取迭代器
            snapshot = elements;
        }

        // 判断是否遍历列表
        public boolean hasNext() {
            return cursor < snapshot.length;
        }

        public boolean hasPrevious() {
            return cursor > 0;
        }

        // 获取下一个元素
        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            if (! hasPrevious())
                throw new NoSuchElementException();
            return (E) snapshot[--cursor];
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor-1;
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code remove}
         *         is not supported by this iterator.
         */
        public void remove() {
            throw new UnsupportedOperationException();
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code set}
         *         is not supported by this iterator.
         */
        public void set(E e) {
            throw new UnsupportedOperationException();
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code add}
         *         is not supported by this iterator.
         */
        public void add(E e) {
            throw new UnsupportedOperationException();
        }

        @Override
        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            Object[] elements = snapshot;
            final int size = elements.length;
            for (int i = cursor; i < size; i++) {
                @SuppressWarnings("unchecked") E e = (E) elements[i];
                action.accept(e);
            }
            cursor = size;
        }
    }
```


**demo**

```java
public class Demo {
	// 线程共享资源类
	private static volatile CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

	public static void main(String[] args) throws InterruptedException {
		list.add("1");
		list.add("2");
		list.add("3");
		
		Thread t1 = new Thread(() -> {
			list.add("4");
			
			list.set(1, "222");
		});
		
		// 在子线程启动之前获取迭代器，生成就数组快照
		Iterator<String> it = list.iterator();
		
		// 启动子线程
		t1.start();
		
		// 等待子线程执行完
		t1.join();
		
		// 迭代元素
		while (it.hasNext()) {
			System.out.println(it.next());
		}
		
	}

}
```

**控制台结果：**
```console
1
2
3
```
