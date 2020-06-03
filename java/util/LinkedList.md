## LinkedList源码解析

#### LinkedList的数据结构
```java
    // 元素个数
    transient int size = 0;

    /**
     * 记录头节点
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * 记录尾节点
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;

    // 节点数据结构
    private static class Node<E> {
        // 保存E类型的数据
        E item;
        // 后节点
        Node<E> next;
        // 前节点
        Node<E> prev;

        // 构造器参数列表遵循前中后，方便阅读
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

#### LinkedList的构造器
```java
    // 无参构造
    public LinkedList() {}

    // 将Collection类型的数据集放到LinkedList集合中
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```

#### LinkedList的实例方法
1. addFirst(E e)：将元素放在首位
```java
    /**
     * Inserts the specified element at the beginning of this list.
     *
     * @param e the element to add
     */
    public void addFirst(E e) {
        linkFirst(e);
    }

    /**
     * Links e as first element.
     */
    private void linkFirst(E e) {
        // 记录旧的list的头节点
        final Node<E> f = first;
        // 生成新节点：由于放在首位，无前节点，所以为null
        final Node<E> newNode = new Node<>(null, e, f);
        // 将头节点设为新节点
        first = newNode;
        // 如果旧的头节点不存在，说明集合为空
        if (f == null)
            // 则新节点即为头节点，也为尾节点
            last = newNode;
        else
            // 如果旧的头节点存在，即需要将其前节点改为新节点
            f.prev = newNode;
        // 元素个数加1
        size++;
        // 修改次数加1(相当于版本号)
        modCount++;
    }
```

2. addLast(E e)：将元素放在末位
```java
    /**
     * Appends the specified element to the end of this list.
     *
     * <p>This method is equivalent to {@link #add}.
     *
     * @param e the element to add
     */
    public void addLast(E e) {
        linkLast(e);
    }

    /**
     * Links e as last element.
     */
    void linkLast(E e) {
        // 记录旧的list的尾节点
        final Node<E> l = last;
        // 生成新节点：由于放在末位，无后节点，所以为null
        final Node<E> newNode = new Node<>(l, e, null);
        // 将尾节点设为新节点
        last = newNode;
        // 如果旧的尾节点不存在，说明集合为空
        if (l == null)
            // 则新节点即为尾节点，也为头节点
            first = newNode;
        else
            // 如果旧的尾巴节点存在，即需要将其后节点改为新节点
            l.next = newNode;
        // 元素个数加1
        size++;
        // 修改次数加1(相当于版本号)
        modCount++;
    }
```

3. add(int index, E element)：指定位置后添加元素
```java
    public void add(int index, E element) {
        // 检查是否在0-size之间，不在抛出IndexOutOfBoundsException
        // 注意：包含0，并不是从1，开始检查，因为size初始值是0
        checkPositionIndex(index);
        // 如果是size大小
        if (index == size)
            // 放末位
            linkLast(element);
        else
            // 在之前插入
            linkBefore(element, node(index));
    }

    void linkBefore(E e, Node<E> succ) {
        // 记录succ节点的前节点，succ在node方法的计算中根本不可能等于null，所以官方这里断言就是这个意思，不需要在if判断了
        // assert succ != null;
        final Node<E> pred = succ.prev;
        // 生成新节点前节点为succ节点的前节点，后节点为succ节点
        final Node<E> newNode = new Node<>(pred, e, succ);
        // succ节点的前节点为新节点
        succ.prev = newNode;
        // 如果succ节点的前节点为null，即succ节点为旧头节点
        if (pred == null)
            // 所以新节点为头节点
            first = newNode;
        else
            // 新节点为pred的后节点
            pred.next = newNode;
        // 元素个数加1
        size++;
        // 修改次数加1(相当于版本号)
        modCount++;
    }

    /**
    * 这就是通过下标来找元素，效率很低，要不断迭代，时间复杂度为O(n)/2
    */
    Node<E> node(int index) {
        // assert isElementIndex(index);
        // 由于是双向的，除以2
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            // 不断迭代寻找
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

4. removeFirst(): 将首位元素移出
```java
    public E removeFirst() {
        final Node<E> f = first;
        // first不存在，即容器是空的，无东西移出抛异常
        if (f == null)
            throw new NoSuchElementException();
        // remove
        return unlinkFirst(f);
    }

    // 由上面的方法可以断言f == first && f != null，无需再判断
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        // 获取首节点存放的值
        final E element = f.item;
        // 获取第二个节点
        final Node<E> next = f.next;
        // 断开首节点存放的值引用
        f.item = null;
        // 首节点后节点也要断开引用，前节点本身就是null
        f.next = null; // help GC
        // 此时首节点应该是原来的第二个节点
        first = next;
        // 如果next=null,说明就只有一个元素，存在一个元素的情况，说明next和last指向同一个对象
        if (next == null)
            // 则last为null
            last = null;
        else
            // 晋升为首节点，确保无前节点
            next.prev = null;
        // 元素个数-1
        size--;
        // 修改次数加1
        modCount++;
        // 返回remove的值
        return element;
    }

```

5. clear():
```java
    public void clear() {
        // Clearing all of the links between nodes is "unnecessary", but:
        // - helps a generational GC if the discarded nodes inhabit
        //   more than one generation
        // - is sure to free memory even if there is a reachable Iterator
        // 从头开始remove
        for (Node<E> x = first; x != null; ) {
            // 获取下一个节点
            Node<E> next = x.next;
            // 当前节点item,next,prev = null
            x.item = null;
            x.next = null;
            x.prev = null;
            // 迭代下一个节点
            x = next;
        }
        // 确保首尾节点为null
        first = last = null;
        // 元素个数为0
        size = 0;
        // 修改次数加1
        modCount++;
    }
```

### 如果说以上的代码能够看得懂，其他的一些普通方法都是可以看得懂的，像peek，poll都应该很简单，下面有个区别，如下所示：

##### 可以看得出来isPositionIndex和isElementIndex有个"="的区别，那是因为node方法是需要类似于通过下标去查找元素，所以下标就和数组下标一致([0, arr.length-1]),这样程序员是更能够去接受的，所以这两边的判断相差一点，别懵，和node方法一点关系没有，下面注释有说明
```java
    // 代码1
    public void add(int index, E element) {
        checkPositionIndex(index);

        // 这里有对size处理，所以node方法还是处理的[0, arr.length-1]，这里的arr只是我将LinkedList比作数组
        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    private void checkPositionIndex(int index) {
        if (!isPositionIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private boolean isPositionIndex(int index) {
        return index >= 0 && index <= size; // 差异
    }


    // 代码2
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }

    private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private boolean isElementIndex(int index) {
        return index >= 0 && index < size; // 差异
    }
    
    
```

## 迭代器
#### 迭代器其实是一个设计模式，java23种设计模式，有兴趣的话大家可以去学习学习，但是迭代器必须要掌握，迭代器必须要依靠数据结构去支持，好，那下面看看人家是怎么实现的。
```java
private class ListItr implements ListIterator<E> {
        // 记录最后一个return的节点
        private Node<E> lastReturned;
        // 记录下一个节点
        private Node<E> next;
        // 记录下一个下标
        private int nextIndex;
        // 记录一下版本号
        private int expectedModCount = modCount;

        // 这个下标意思是从哪个位置开始迭代
        ListItr(int index) {
            // assert isPositionIndex(index);
            // 在构造迭代器之前人家已经确保了index是属于[0, size]
            // index == size说明没有元素，别忘了下标类似数组，没啥迭代的，在这里为什么要用isPositionIndex判断而不用isElementIndex，其实两种都行，然而迭代器的代码实现就需要改动，有能力的可以自己写
            // 有人有可能会纠结这种情况
            // 如果有五个元素【1,2,3,4,5】所对应的下标为【0,1,2,3,4】
            // 假设我想从下标从4开始迭代,node()=last，即是5元素 
            // 如果下标从5开始，next=null
            // 这里其实指定下标的这个元素还未开始迭代，可以看得出来next是last值的，而且下面hasNext()为true就能说明。
            next = (index == size) ? null : node(index);
            // 将下标赋值
            nextIndex = index;
        }

        // 主要是判断有没有下一个
        public boolean hasNext() {
            return nextIndex < size;
        }

        // 后移
        public E next() {
            // 检查一下版本是否一致，不一致则出现了并发问题
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();
            // 保存最后一次返回的节点
            lastReturned = next;
            // 记录下一个节点
            next = next.next;
            // 记录下一个下标
            nextIndex++;
            // 返回节点值
            return lastReturned.item;
        }

        public boolean hasPrevious() {
            return nextIndex > 0;
        }

        // 前移
        public E previous() {
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();
            // 下一个没有元素说明已经迭代完成，前移肯定就是最后一个元素
            lastReturned = next = (next == null) ? last : next.prev;
            nextIndex--;
            return lastReturned.item;
        }

        public int nextIndex() {
            return nextIndex;
        }

        public int previousIndex() {
            return nextIndex - 1;
        }

        public void remove() {
            checkForComodification();
            if (lastReturned == null)
                throw new IllegalStateException();

            Node<E> lastNext = lastReturned.next;
            unlink(lastReturned);
            if (next == lastReturned)
                next = lastNext;
            else
                nextIndex--;
            lastReturned = null;
            expectedModCount++;
        }

        public void set(E e) {
            if (lastReturned == null)
                throw new IllegalStateException();
            checkForComodification();
            lastReturned.item = e;
        }

        public void add(E e) {
            checkForComodification();
            lastReturned = null;
            if (next == null)
                linkLast(e);
            else
                linkBefore(e, next);
            nextIndex++;
            expectedModCount++;
        }

        // 上面方法还是很简单的，咱看这个
        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            while (modCount == expectedModCount && nextIndex < size) {
                // Consumer函数式接口，可以实现我们自己的逻辑，这种代码写法可以学习一下，下面我也贴出例子，但是由于LinkedList线程并不安全，modCount == expectedModCount和checkForComodification都是对版本的检验
                action.accept(next.item);
                lastReturned = next;
                next = next.next;
                nextIndex++;
            }
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

### 例子
```java
package container.collection.linkedlist;

import java.util.LinkedList;
import java.util.ListIterator;

/**
 * 
 * @author 王浩
 * @Date 2020年5月18日
 * @Theme: LinkedList case
 * @VM_Args: 
 * @Description:
 */
public class Demo01 {

	public static void main(String[] args) {
		LinkedList<String> l = new LinkedList<String>();
		l.add("1");
		l.add("2");
		l.add("3");
		
		LinkedList<String> l2 = new LinkedList<String>();
		ListIterator<String> t = l.listIterator(0);
		t.forEachRemaining(e -> {
			l2.add(e);
		});
		
		System.out.println(l2); // [1, 2, 3]
	}
}

```


```java
package java.util.function;

import java.util.Objects;

/**
 * Represents an operation that accepts a single input argument and returns no
 * result. Unlike most other functional interfaces, {@code Consumer} is expected
 * to operate via side-effects.
 *
 * <p>This is a <a href="package-summary.html">functional interface</a>
 * whose functional method is {@link #accept(Object)}.
 *
 * @param <T> the type of the input to the operation
 *
 * @since 1.8
 */
@FunctionalInterface
public interface Consumer<T> {

    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);

    /**
     * Returns a composed {@code Consumer} that performs, in sequence, this
     * operation followed by the {@code after} operation. If performing either
     * operation throws an exception, it is relayed to the caller of the
     * composed operation.  If performing this operation throws an exception,
     * the {@code after} operation will not be performed.
     *
     * @param after the operation to perform after this operation
     * @return a composed {@code Consumer} that performs in sequence this
     * operation followed by the {@code after} operation
     * @throws NullPointerException if {@code after} is null
     */
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}

```
