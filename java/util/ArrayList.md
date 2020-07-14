## ArrayList源码解析

#### ArrayList的数据结构
```java
/**
* Default initial capacity.
* 默认初始容量
*/
private static final int DEFAULT_CAPACITY = 10;

/**
* Shared empty array instance used for empty instances.
* 空的共享数组
*/
private static final Object[] EMPTY_ELEMENTDATA = {};

/**
* Shared empty array instance used for default sized empty instances. We
* distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
* first element is added.
*/
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

// 底层数组
transient Object[] elementData;

// 元素个数
private int size;
```

#### ArrayList的构造器
```java
// 指定数组容量构造器
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        // 指定大于0，直接创建指定容量数组
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        // 用常量的空数组
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        // 抛异常
        throw new IllegalArgumentException("Illegal Capacity: "+
                                            initialCapacity);
    }
}

// 无参构造
public ArrayList() {
    // 使用初始空数组
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 构造集合类元素
public ArrayList(Collection<? extends E> c) {
    // 将对应元素转成数组
    elementData = c.toArray();
    // 比较并赋值size
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        // 如果不是Object数组类型，就通过Arrays.copyOf进行转化
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // 如果集合类无元素，给空数组，与上面无参构造给空数组有所区别
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

#### ArrayList的实例方法
1. get(int index)：指定位置存放
```java
public E get(int index) {
    // 先进行下标范围检查
    rangeCheck(index);
    // 检查没有问题就进行根据索引获取元素
    return elementData(index);
}

private void rangeCheck(int index) {
    // 如果是大于或等于元素个数，就抛出异常，注意这里并没有和数组长度比较
    // 所以可以推断元素肯定是优先放在低位，“坐无缺席”。
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

E elementData(int index) {
    // 直接操作底层数组获取元素
    return (E) elementData[index];
}
```

2. set(int index, E element)：指定位置更新元素
```java
public E set(int index, E element) {
    // 先检查下标
    rangeCheck(index);
    // 获取旧值
    E oldValue = elementData(index);
    // 放入新值
    elementData[index] = element;
    // 返回旧值
    return oldValue;
}
```

3. add(E e)： 添加新元素
```java
public boolean add(E e) {
    // 在size基础上+1进入ensureCapacityInternal方法
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 在size位置添加新的元素 后!!! size再+1
    elementData[size++] = e;
    return true;
}

// 传入最小容量
private void ensureCapacityInternal(int minCapacity) {
    // 如果使用无参构造
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 和值取一个最大值
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    // 传入
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    // 结构发生改变 +1
    modCount++;

    // overflow-conscious code
    // 如果需要的最小元素个数比实际的数组容量大，就需要数组扩容
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

// 扩容方法
private void grow(int minCapacity) {
    // overflow-conscious code
    // 获取数组容量
    int oldCapacity = elementData.length;
    // 计算新数组容量，1.5倍旧的数组
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        // 如果计算完发现新的数组容量还是比所需要的小，最小容量直接赋值
        // 这里大多数是在第一次扩容的时候
        // elementData是空数组，长度为0，新的扩容1.5还是0
        // 元素都是一个个增的情况，所以第一次会走
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        // 如果比最大值大
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    // 扩容新的数组长度为新的容量
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    // oom
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

4. add(int index, E element)： 指定下标添加新元素
```java
public void add(int index, E element) {
    // 下标检查
    rangeCheckForAdd(index);
    // 结构+1，查看需不需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 将index及后面的元素后移一位
    System.arraycopy(elementData, index, elementData, index + 1,
                        size - index);
    // 在index位置设置元素
    elementData[index] = element;
    // 元素个数+1
    size++;
}

private void rangeCheckForAdd(int index) {
    // index = size 并没有做check，如果是size，就是在所有元素最后
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

5. remove(int index)：指定下标删除
```java
public E remove(int index) {
    // 检查下标
    rangeCheck(index);
    // 结构+1
    modCount++;
    // 获取该位置的旧值
    E oldValue = elementData(index);
    // 后面的元素需要前移，此计算计算的是要移动的元素的个数
    int numMoved = size - index - 1;
    if (numMoved > 0)
        // 进行前移
        System.arraycopy(elementData, index+1, elementData, index,
                            numMoved);
    // 先对元素个数-1，再将最后一个元素删除，该位置留的是移动之前最后一位元素的值
    elementData[--size] = null; // clear to let GC do its work
    // 返回旧值
    return oldValue;
}
```

6. remove(Object o)：指定元素删除
```java
public boolean remove(Object o) {
    // 只能删除第一个遇到的元素，看见return没有
    // o为null的情况
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            // 通过equals比较进行删除，不是引用相等删除。删除的是第一次遇到。
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

// 根据下标快速删除
private void fastRemove(int index) {
    // 结构+1
    modCount++;
    // 元素移动个数
    int numMoved = size - index - 1;
    if (numMoved > 0)
    // 移动
        System.arraycopy(elementData, index+1, elementData, index,
                            numMoved);
    // 和上面一样
    elementData[--size] = null; // clear to let GC do its work
}
```

#### ArrayList的迭代器


#### fail-fast(快速失败)机制