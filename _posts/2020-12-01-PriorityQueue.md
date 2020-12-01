---
author: renfakai
layout: post
title: PriorityQueue 源码翻译的解读
date: 2020-12-01
categories: Java
tags: [java]
description: PriorityQueue 源码翻译的解读
---

认真读源码

##  认真读源码

### 注释部分

```java 

/**
 * An unbounded priority {@linkplain Queue queue} based on a priority heap.
 * 无界优先级队列基于优先堆
 * The elements of the priority queue are ordered according to their
 * 优先级队列会根据Comparable或比较器进行排序，并且提供了相应的构造函数,依赖于那个构造函数
 * {@linkplain Comparable natural ordering}, or by a {@link Comparator}
 * provided at queue construction time, depending on which constructor is
 * 优先队列不允许空元素
 * used.  A priority queue does not permit {@code null} elements.
 * 优先级队列依赖自然排序当对象没有实现comparable接口时
 * A priority queue relying on natural ordering also does not permit
 * insertion of non-comparable objects (doing so may result in
 * 结果可能会导致ClassCastException异常
 * {@code ClassCastException}).
 * 队列的头是按照指定排序的最后一个原属，如果多个元素尝试最后一个值，这个头可能会从其中选择一个
 * <p>The <em>head</em> of this queue is the <em>least</em> element
 * with respect to the specified ordering.  If multiple elements are
 * 尝试次数减少会被任意打破
 * tied for least value, the head is one of those elements -- ties are
 * 队列的恢复操作 poll remove peek 允许从队列头获取元素
 * broken arbitrarily.  The queue retrieval operations {@code poll},
 * {@code remove}, {@code peek}, and {@code element} access the
 * element at the head of the queue.
 *
 * 优先队列是无界的，但是一个内部主要容量的数组来保存数据
 * <p>A priority queue is unbounded, but has an internal
 * <i>capacity</i> governing the size of an array used to store the
 * 他总是和队列一样大的尺寸
 * elements on the queue.  It is always at least as large as the queue
 * 元素被增加到队列时候，会自动扩容，扩容的策略不指定
 * size.  As elements are added to a priority queue, its capacity
 * grows automatically.  The details of the growth policy are not
 * specified.
 * 类的迭代实现了Iterator协议
 * <p>This class and its iterator implement all of the
 * <em>optional</em> methods of the {@link Collection} and {@link
 * 迭代器提供了一些列的方法来保证队列以任意特定顺序
 * Iterator} interfaces.  The Iterator provided in method {@link
 * #iterator()} is <em>not</em> guaranteed to traverse the elements of
 * 如果你要遍历排序，考虑使用Arrays.sort()
 * the priority queue in any particular order. If you need ordered
 * traversal, consider using {@code Arrays.sort(pq.toArray())}.
 *
 * 温馨提示队列没有使用同步
 * <p><strong>Note that this implementation is not synchronized.</strong>
 * 对线程不应该同时操作优先级队列
 * Multiple threads should not access a {@code PriorityQueue}
 * instance concurrently if any of the threads modifies the queue.
 * 如果使用多线程，建议使用PriorityBlockingQueue
 * Instead, use the thread-safe {@link
 * java.util.concurrent.PriorityBlockingQueue} class.
 *
 * 实现提供O(log(n))时间复杂度对进队列和出队列
 * <p>Implementation note: this implementation provides
 * O(log(n)) time for the enqueuing and dequeuing methods
 * ({@code offer}, {@code poll}, {@code remove()} and {@code add});
 * linear time for the {@code remove(Object)} and {@code contains(Object)}
 * methods; and constant time for the retrieval methods
 * ({@code peek}, {@code element}, and {@code size}).
 *
 * <p>This class is a member of the
 * <a href="{@docRoot}/../technotes/guides/collections/index.html">
 * Java Collections Framework</a>.
 *
 * @since 1.5
 * @author Josh Bloch, Doug Lea
 * @param <E> the type of elements held in this collection
 */

```

源码阅读

```
// 抽象队列和序列化
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {

    // 序列化
    private static final long serialVersionUID = -7720805057305804111L;

    // 默认size
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * 优先级队列使用的平衡二叉堆
     * Priority queue represented as a balanced binary heap: the two
     * 对于队列中第n的元素的两个子节点分别处于 2*n + 1 和 2*（n+1）c处
     * children of queue[n] are queue[2*n+1] and queue[2*(n+1)].  The
     * 优先级队列根据排序器进行排序，活着自然排序
     * priority queue is ordered by comparator, or by the elements'
     * 如果排序器为空，迭代对上的每一个节点 
     * natural ordering, if comparator is null: For each node n in the
     * heap and each descendant d of n, n <= d.  The element with the
     * 原属最小值为第一个，如果队列不为空的话
     * lowest value is in queue[0], assuming the queue is nonempty.
     * 非私有来保证内部类访问
     */
    transient Object[] queue; // non-private to simplify nested class access

    /**
     * The number of elements in the priority queue.
     * 优先队列的数量
     */
    private int size = 0;

    /**
     * The comparator, or null if priority queue uses elements'
     * natural ordering.
     * 排序器，如果为空的话使用自然排序
     */
    private final Comparator<? super E> comparator;

    /**
    * 优先队列结构上被修改的次数，版本概念
     * The number of times this priority queue has been
     * <i>structurally modified</i>.  See AbstractList for gory details.
     */
    transient int modCount = 0; // non-private to simplify nested class access

    /**
     * 创建一个特定大小(11)的队列， 并根据字段排序
     * Creates a {@code PriorityQueue} with the default initial
     * capacity (11) that orders its elements according to their
     * {@linkplain Comparable natural ordering}.
     */
    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    /**
     * 创建指定大小并自然排序的队列
     * Creates a {@code PriorityQueue} with the specified initial
     * capacity that orders its elements according to their
     * {@linkplain Comparable natural ordering}.
     * 队列初始化大小
     * @param initialCapacity the initial capacity for this priority queue
     * 异常
     * @throws IllegalArgumentException if {@code initialCapacity} is less
     *         than 1
     */
    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    /**
     * 创建一个初始化的队列，并且根据排序器进行排序
     * Creates a {@code PriorityQueue} with the default initial capacity and
     * whose elements are ordered according to the specified comparator.
     * 排序器
     * @param  comparator the comparator that will be used to order this
     *         priority queue.  If {@code null}, the {@linkplain Comparable
     *         natural ordering} of the elements will be used.
     * @since 1.8
     */
    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }

    /**
     * 根据给定的大小创建队列并使用排序器
     * Creates a {@code PriorityQueue} with the specified initial capacity
     * that orders its elements according to the specified comparator.
     * 队列初始化的大小
     * @param  initialCapacity the initial capacity for this priority queue
     * 排序器
     * @param  comparator the comparator that will be used to order this
     *         priority queue.  If {@code null}, the {@linkplain Comparable
     *         natural ordering} of the elements will be used.
     * 如果大小小于1 出现非法参数异常
     * @throws IllegalArgumentException if {@code initialCapacity} is
     *         less than 1
     */
    public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // but continues for 1.5 compatibility
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }

    /**
     * 将一个集合放到队列里
     * Creates a {@code PriorityQueue} containing the elements in the
     * 如果集合集成了SortedSet活着PriorityQueue 队列的排序使用相同策略
     * specified collection.  If the specified collection is an instance of
     * a {@link SortedSet} or is another {@code PriorityQueue}, this
     * priority queue will be ordered according to the same ordering.
     * 换句话说，这个优先级队列会使用自然排序
     * Otherwise, this priority queue will be ordered according to the
     * {@linkplain Comparable natural ordering} of its elements.
     * 
     * 需要被加入的集合
     * @param  c the collection whose elements are to be placed
     *         into this priority queue
     * 类型转换异常   如果指定集合的原属不能于队列其他元素进行比的话
     * @throws ClassCastException if elements of the specified collection
     *         cannot be compared to one another according to the priority
     *         queue's ordering
     * 如果指定集合为空，活着内部数据元素为空的话
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    @SuppressWarnings("unchecked")
    public PriorityQueue(Collection<? extends E> c) {
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            initElementsFromCollection(ss);
        }
        else if (c instanceof PriorityQueue<?>) {
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        else {
            this.comparator = null;
            initFromCollection(c);
        }
    }

    /**
     * 创建一个优先级别的队列这个队列根据相同的优先级队列
     * Creates a {@code PriorityQueue} containing the elements in the
     * specified priority queue.  This priority queue will be
     * ordered according to the same ordering as the given priority
     * queue.
     * 优先级队列集合
     * @param  c the priority queue whose elements are to be placed
     *         into this priority queue
     * 类型转换异常，对于排序不能对比
     * @throws ClassCastException if elements of {@code c} cannot be
     *         compared to one another according to {@code c}'s
     *         ordering
     * 如果指定队列为空活着内部元素为空
     * @throws NullPointerException if the specified priority queue or any
     *         of its elements are null
     */
    @SuppressWarnings("unchecked")
    public PriorityQueue(PriorityQueue<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initFromPriorityQueue(c);
    }

    /**
     * 创建优先队列包含指定排序
     * Creates a {@code PriorityQueue} containing the elements in the
     * 排序使用相同策略
     * specified sorted set.   This priority queue will be ordered
     * according to the same ordering as the given sorted set.
     *
     * @param  c the sorted set whose elements are to be placed
     *         into this priority queue
     * @throws ClassCastException if elements of the specified sorted
     *         set cannot be compared to one another according to the
     *         sorted set's ordering
     * @throws NullPointerException if the specified sorted set or any
     *         of its elements are null
     */
    @SuppressWarnings("unchecked")
    public PriorityQueue(SortedSet<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initElementsFromCollection(c);
    }

    // 初始化优先级队列
    private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
        // 如果底层一样，进行指针覆值
        if (c.getClass() == PriorityQueue.class) {
            this.queue = c.toArray();
            this.size = c.size();
        } else {
            // 按照集合覆值
            initFromCollection(c);
        }
    }

    private void initElementsFromCollection(Collection<? extends E> c) {
       // 将集合转换成数组
        Object[] a = c.toArray();
        // 如果数据底层不是object数组，复制数据
        // If c.toArray incorrectly doesn't return Object[], copy it.
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, a.length, Object[].class);
        // 对数组进行校验，空原属校验
        int len = a.length;
        if (len == 1 || this.comparator != null)
            for (int i = 0; i < len; i++)
                if (a[i] == null)
                    throw new NullPointerException();
        this.queue = a;
        this.size = a.length;
    }

    /**
     * 初始化集合
     * Initializes queue array with elements from the given Collection.
     *
     * @param c the collection
     */
    private void initFromCollection(Collection<? extends E> c) {
        initElementsFromCollection(c);
        // 堆话
        heapify();
    }

    /**
     * 数组最大可分配
     * The maximum size of array to allocate.
     * 一个Vms对数组保留一些空间
     * Some VMs reserve some header words in an array.
     * 尝试分配最大数组结果
     * Attempts to allocate larger arrays may result in
     * 如果超过Vm限制最大限制会出现OOM
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * Increases the capacity of the array.
     * 扩容
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // 旧的大小
        int oldCapacity = queue.length;
        // Double size if small; else grow by 50%
        // 如果size特别小，使用 old + old + 2
        // 大于64 直接double
        int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                         (oldCapacity + 2) :
                                         (oldCapacity >> 1));
        // overflow-conscious code
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        queue = Arrays.copyOf(queue, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        // 小于0 已经oom了
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

    /**
     * 插入指定元素到队列中
     * Inserts the specified element into this priority queue.
     * 返回是否加入成功
     * @return {@code true} (as specified by {@link Collection#add})
     * 如果指定元素不能排序在队列排序的话会出现异常
     * @throws ClassCastException if the specified element cannot be
     *         compared with elements currently in this priority queue
     *         according to the priority queue's ordering
     * 元素为空的话会出现空指针异常
     * @throws NullPointerException if the specified element is null
     */
    public boolean add(E e) {
        return offer(e);
    }

    /**
     * Inserts the specified element into this priority queue.
     *
     * @return {@code true} (as specified by {@link Queue#offer})
     * @throws ClassCastException if the specified element cannot be
     *         compared with elements currently in this priority queue
     *         according to the priority queue's ordering
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        // 元素为空空指针异常
        if (e == null)
            throw new NullPointerException();
        // version++ 
        modCount++;
         
       // 队列满了，扩容
        int i = size;
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        if (i == 0)
            queue[0] = e;
        else
       // 上浮
            siftUp(i, e);
        return true;
    }

    @SuppressWarnings("unchecked")
    public E peek() {
        // 没有数据的返回空，否则获取最小值
        return (size == 0) ? null : (E) queue[0];
    }

    // 进行迭代对比，查找角标
    private int indexOf(Object o) {
        if (o != null) {
            for (int i = 0; i < size; i++)
                if (o.equals(queue[i]))
                    return i;
        }
        return -1;
    }

    /**
     * 从队列中移除单个元素，如果可以的话
     * Removes a single instance of the specified element from this queue,
     * 更常用的是像对比，如果队列中包含多个元素
     * if it is present.  More formally, removes an element {@code e} such
     * that {@code o.equals(e)}, if this queue contains one or more such
     * elements.  Returns {@code true} if and only if this queue contained
     * 当着队列包含这个元素的时候
     * the specified element (or equivalently, if this queue changed as a
     * result of the call).
     * 如果原属在队列中，并且调用成功的话会返回true
     * @param o element to be removed from this queue, if present
     * @return {@code true} if this queue changed as a result of the call
     */
    public boolean remove(Object o) {
        int i = indexOf(o);
        if (i == -1)
            return false;
        else {
            removeAt(i);
            return true;
        }
    }

    /**
     * 如果医用相等的话使用迭代器remove
     * Version of remove using reference equality, not equals.
     * Needed by iterator.remove.
     *
     * @param o element to be removed from this queue, if present
     * @return {@code true} if removed
     */
    boolean removeEq(Object o) {
        for (int i = 0; i < size; i++) {
            if (o == queue[i]) {
                removeAt(i);
                return true;
            }
        }
        return false;
    }

    /**
     * 查找角标
     * Returns {@code true} if this queue contains the specified element.
     * More formally, returns {@code true} if and only if this queue contains
     * at least one element {@code e} such that {@code o.equals(e)}.
     *
     * @param o object to be checked for containment in this queue
     * @return {@code true} if this queue contains the specified element
     */
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }

    /**
     * 返回一个数组包含队列中的所有原属
     * Returns an array containing all of the elements in this queue.
     * 这些元素没有特殊的排序
     * The elements are in no particular order.
     * 这个返回会很安全，因为数据不会维持队列的引用，换句话说，这个方法会分配新的数组
     * <p>The returned array will be "safe" in that no references to it are
     * maintained by this queue.  (In other words, this method must allocate
     * 调用修改返回数组是自由的
     * a new array).  The caller is thus free to modify the returned array.
     * 
     * <p>This method acts as bridge between array-based and collection-based
     * APIs.
     *
     * @return an array containing all of the elements in this queue
     */
    public Object[] toArray() {
        return Arrays.copyOf(queue, size);
    }

    /**
     * 返回一个数组会包含所有队列中的所有元素
     * Returns an array containing all of the elements in this queue; the
     * 指定数组允许时type?
     * runtime type of the returned array is that of the specified array.
     * 返回的数组不会包含特殊的排序
     * The returned array elements are in no particular order.
     * 如果这个队列适合指定数组，会将数据放到里面
     * If the queue fits in the specified array, it is returned therein.
     * 其他，如果一个新数组在运行时允许被分配指定数组
     * Otherwise, a new array is allocated with the runtime type of the
     * specified array and the size of this queue.
     * 如果队列适合指定的数组并有剩余空间
     * <p>If the queue fits in the specified array with room to spare
     * 如果数组有更多的元素比队列
     * (i.e., the array has more elements than the queue), the element in
     * 紧接集合结束后的数组设置为null
     * the array immediately following the end of the collection is set to
     * {@code null}.
     * 像toArray方法，这个方法被桥接基于集合的API
     * <p>Like the {@link #toArray()} method, this method acts as bridge between
     * 更远的是，这个方法在运行时允许精确控制将类型输出到数组中
     * array-based and collection-based APIs.  Further, this method allows
     * precise control over the runtime type of the output array, and may,
     * 在某些情况下，可用于节省分配成本
     * under certain circumstances, be used to save allocation costs.
     * 如果队列中仅包含字符串
     * <p>Suppose {@code x} is a queue known to contain only strings.
     * 可用使用下面的code进行分配数组数据
     * The following code can be used to dump the queue into a newly
     * allocated array of {@code String}:
     *
     *  <pre> {@code String[] y = x.toArray(new String[0]);}</pre>
     * 
     * Note that {@code toArray(new Object[0])} is identical in function to
     * {@code toArray()}.
     * 数组在队列中被加载，如果足够大的话，换句话说，一个新数组在运行的目的使用允许被分配
     * @param a the array into which the elements of the queue are to
     *          be stored, if it is big enough; otherwise, a new array of the
     *          same runtime type is allocated for this purpose.
     * 返回一个包含队列中所有数组的数组
     * @return an array containing all of the elements in this queue
     * 如果指定数组运行是的超型
     * @throws ArrayStoreException if the runtime type of the specified array
     *         is not a supertype of the runtime type of every element in
     *         this queue
     * 如果指定的数组为空的话
     * @throws NullPointerException if the specified array is null
     */
    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        // 数组的size
        final int size = this.size;
        // 如果数组长度小于队列长度
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
             // 创建新的数组
            return (T[]) Arrays.copyOf(queue, size, a.getClass());
        // 数组copy
        System.arraycopy(queue, 0, a, 0, size);
       
        // ?
        if (a.length > size)
            a[size] = null;
        return a;
    }

    /**
     * 返回队列中的所有迭代器
     * Returns an iterator over the elements in this queue. The iterator
     * 返回的元素没有特俗的排序
     * does not return the elements in any particular order.
     * 返回迭代队列中的迭代器
     * @return an iterator over the elements in this queue
     */
    public Iterator<E> iterator() {
        return new Itr();
    }

    private final class Itr implements Iterator<E> {
        /**
         * 游标
         * Index (into queue array) of element to be returned by
         * subsequent call to next.
         */
        private int cursor = 0;

        /**
         * 最近一次调用next返回的元素索引
         * Index of element returned by most recent call to next,
         * 除非这个元素属于forgetMeNot列表
         * unless that element came from the forgetMeNot list.
         * 如果这个元素被删除活着移除返回-1
         * Set to -1 if element is deleted by a call to remove.
         */
        private int lastRet = -1;

        /**
         * 从元素的未访问部分移出的元素队列
         * A queue of elements that were moved from the unvisited portion of
         * 迭代期间的删除由于“不幸”元素而使堆进入访问的部分
         * the heap into the visited portion as a result of "unlucky" element
         * removals during the iteration.  (Unlucky element removals are those
         * 我们从迭代器中访问所有元素完成
         * that require a siftup instead of a siftdown.)  We must visit all of
         * the elements in this list to complete the iteration.  We do this
         * after we've completed the "normal" iteration.
         * 我们期待更多的迭代器，即使涉及到搬迁
         * We expect that most iterations, even those involving removals,
         * 这个会加载元素到这个Filed中
         * will not need to store elements in this field.
         */
        private ArrayDeque<E> forgetMeNot = null;

        /**
         * 对下一个的最新调用返回的元素 iff 如果元素是从forgetMeNot列表中绘制的
         * Element returned by the most recent call to next iff that
         * element was drawn from the forgetMeNot list.
         */
        private E lastRetElt = null;

        /**
         * 如果迭代器的修改值与队列中的值一样
         * The modCount value that the iterator believes that the backing
         * 如果期待被修改，迭代器会监测到被修改
         * Queue should have.  If this expectation is violated, the iterator
         * has detected concurrent modification.
         */
        private int expectedModCount = modCount;

        public boolean hasNext() {
        // 游标未到达size
            return cursor < size ||
                (forgetMeNot != null && !forgetMeNot.isEmpty());
        }

        @SuppressWarnings("unchecked")
        public E next() {
            // 多线程修改了
            if (expectedModCount != modCount)
                throw new ConcurrentModificationException();
            // 游标问题
            if (cursor < size)
                return (E) queue[lastRet = cursor++];
            // 最后等于 -1 
            if (forgetMeNot != null) {
                lastRet = -1;
                // 返回数据
                lastRetElt = forgetMeNot.poll();
                if (lastRetElt != null)
                    return lastRetElt;
            }
            throw new NoSuchElementException();
        }

        public void remove() {
             // 多线程修改了
            if (expectedModCount != modCount)
                throw new ConcurrentModificationException();
            if (lastRet != -1) {
                // 内部类角标问题
                E moved = PriorityQueue.this.removeAt(lastRet);
                lastRet = -1;
                if (moved == null)
                    cursor--;
                else {
                    // forget one 
                    if (forgetMeNot == null)
                        forgetMeNot = new ArrayDeque<>();
                    forgetMeNot.add(moved);
                }
            } else if (lastRetElt != null) {
                PriorityQueue.this.removeEq(lastRetElt);
                lastRetElt = null;
            } else {
                throw new IllegalStateException();
            }
            expectedModCount = modCount;
        }
    }

    // size 
    public int size() {
        return size;
    }

    /**
     * Removes all of the elements from this priority queue.
     * The queue will be empty after this call returns.
     */
    public void clear() {
        // verison 增加
        modCount++;
        // clear all
        for (int i = 0; i < size; i++)
            queue[i] = null;
        size = 0;
    }

    @SuppressWarnings("unchecked")
    public E poll() {
        // 数组为空，返回空
        if (size == 0)
            return null;
        // size减少
        int s = --size;
        // 版本增加
        modCount++;
        // 队列最新数据
        E result = (E) queue[0];
        // 最后的一个数据
        E x = (E) queue[s];
        // clear it 
        queue[s] = null;
        if (s != 0)
        // 下沉
            siftDown(0, x);
        return result;
    }

    /**
     * 删除队列中第i角标的元素
     * Removes the ith element from queue.
     *  正常情况下这个下这个方法会移除原属到i-1
     * Normally this method leaves the elements at up to i-1,
     * 包含不可接触的，在这些情况下，他会返回空
     * inclusive, untouched.  Under these circumstances, it returns
     * 偶尔，为了保持堆不变
     * null.  Occasionally, in order to maintain the heap invariant,
     * 必须交换最后一个元素与较前的元素
     * it must swap a later element of the list with one earlier than
     * 在这些情况下，这个方法会返回之前在列表最后并且现在在i 之前的
     * i.  Under these circumstances, this method returns the element
     * that was previously at the end of the list and is now at some
     * 这个被迭代器使用，移除避免丢失遍历元素
     * position before i. This fact is used by iterator.remove so as to
     * avoid missing traversing elements.
     */
    @SuppressWarnings("unchecked")
    private E removeAt(int i) {
        // assert i >= 0 && i < size;
        modCount++;
        int s = --size;
        if (s == i) // removed last element
            queue[i] = null;
        else {
            E moved = (E) queue[s];
            queue[s] = null;
            siftDown(i, moved);
            if (queue[i] == moved) {
                siftUp(i, moved);
                if (queue[i] != moved)
                    return moved;
            }
        }
        return null;
    }

    /**
     * 插入一个元素x在k位置，维持堆内存不变促使x在树上上浮直到它与父节点比较活着root
     * Inserts item x at position k, maintaining heap invariant by
     * promoting x up the tree until it is greater than or equal to
     * its parent, or is the root.
     * 简化并加快强制和比较
     * To simplify and speed up coercions and comparisons. the
     * 比较接口和比较器是在不用的方法中定义的
     * Comparable and Comparator versions are separated into different
     * methods that are otherwise identical. (Similarly for siftDown.)
     *
     * @param k the position to fill
     * @param x the item to insert
     */
    private void siftUp(int k, E x) {
        if (comparator != null)
        // 比较器上浮
            siftUpUsingComparator(k, x);
        else
        // 自然排序上浮
            siftUpComparable(k, x);
    }

    @SuppressWarnings("unchecked")
    private void siftUpComparable(int k, E x) {
        // x -> 转超型 需要对比的数据
        Comparable<? super E> key = (Comparable<? super E>) x;
      
        while (k > 0) {
            // 如果角标k>0，查找父亲节点
            int parent = (k - 1) >>> 1;
            // 拿到父亲节点数据
            Object e = queue[parent];
            // 自然对比
            if (key.compareTo((E) e) >= 0)
                break;
            // 父节点数据转移到k节点
            queue[k] = e;
            // 父节点留空，将新数据放到父节点
            k = parent;
        }
        // 第k位置插入传入的数据
        queue[k] = key;
    }

    @SuppressWarnings("unchecked")
    private void siftUpUsingComparator(int k, E x) {
        // 如果角标k>0，查找父亲节点
        while (k > 0) {
            // 如果角标k>0，查找父亲节点
            // (n-1)/2
            int parent = (k - 1) >>> 1;
            // 拿到父亲节点数据
            Object e = queue[parent];
            // 对比
            if (comparator.compare(x, (E) e) >= 0)
                break;
            // 父亲节点移到k节点
            queue[k] = e;
            
            // index = 父节点
            k = parent;
        }
        queue[k] = x;
    }

    /**
     * 插入一个元素x在角标k上，维持堆内存不变反复将x降级到树上，直到它比子节点或者叶子节点
     * Inserts item x at position k, maintaining heap invariant by
     * demoting x down the tree repeatedly until it is less than or
     * equal to its children or is a leaf.
     *
     * @param k the position to fill
     * @param x the item to insert
     */
    private void siftDown(int k, E x) {
        if (comparator != null)
            // 比较器下沉
            siftDownUsingComparator(k, x);
        else 
            // 自然下沉
            siftDownComparable(k, x);
    }

    @SuppressWarnings("unchecked")
    private void siftDownComparable(int k, E x) {
        // 超型
        Comparable<? super E> key = (Comparable<? super E>)x;
        
        // 使用size找到中间节点，如果没有叶子节点重新迭代
        int half = size >>> 1;        // loop while a non-leaf
       
        // k小于half
        while (k < half) {
            // 获取左节点
            int child = (k << 1) + 1; // assume left child is least
           // 2n+1 
            Object c = queue[child];
            // 右节点
            int right = child + 1;
            //  右节点小于sie,并且左节点大于右节点
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            // 数据等于右节点
                c = queue[child = right];
            // 数据跟有右节点比较
            if (key.compareTo((E) c) <= 0)
            // 小于等于跳出
                break;
            // 左节点大于右节点，并且数据还要大于右节点
            // 第K位置等于有节点
            queue[k] = c;
            //   c = queue[child = right]; 右节点，左节点
            k = child;
        }
        // 放到k坐标
        queue[k] = key;
    }

    // 排序方式使用排序器
    @SuppressWarnings("unchecked")
    private void siftDownUsingComparator(int k, E x) {
        int half = size >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                comparator.compare((E) c, (E) queue[right]) > 0)
                c = queue[child = right];
            if (comparator.compare(x, (E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = x;
    }

    /**
     * 稳定整个二叉树的堆话
     * Establishes the heap invariant (described above) in the entire tree,
     * 假设没有排序元素之前被调用
     * assuming nothing about the order of the elements prior to the call.
     */
    @SuppressWarnings("unchecked")
    private void heapify() {
        // 从数组中间开始下沉
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }

    /**
     * 返回队列元素的比较器
     * Returns the comparator used to order the elements in this
     * 如果队列的元素都使用它来进行排序的话
     * queue, or {@code null} if this queue is sorted according to
     * the {@linkplain Comparable natural ordering} of its elements.
     * 队列排序的比较器
     * @return the comparator used to order this queue, or
     *         {@code null} if this queue is sorted according to the
     *         natural ordering of its elements
     */
    public Comparator<? super E> comparator() {
        return comparator;
    }

    /**
     * 保存队列的序列化
     * Saves this queue to a stream (that is, serializes it).
     *
     * @serialData The length of the array backing the instance is
     *             emitted (int), followed by all of its elements
     *             (each an {@code Object}) in the proper order.
     * @param s the stream
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // Write out element count, and any hidden stuff
        s.defaultWriteObject();

        // Write out array length, for compatibility with 1.5 version
        s.writeInt(Math.max(2, size + 1));

        // Write out all elements in the "proper order".
        for (int i = 0; i < size; i++)
            s.writeObject(queue[i]);
    }

    /**
     * 数据的反序列化
     * Reconstitutes the {@code PriorityQueue} instance from a stream
     * (that is, deserializes it).
     *
     * @param s the stream
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in (and discard) array length
        s.readInt();

        SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, size);
        queue = new Object[size];

        // Read in all elements.
        for (int i = 0; i < size; i++)
            queue[i] = s.readObject();

        // Elements are guaranteed to be in "proper order", but the
        // spec has never explained what that might be.
        // 堆话
        heapify();
    }

    /**
     * Creates a <em><a href="Spliterator.html#binding">late-binding</a></em>
     * and <em>fail-fast</em> {@link Spliterator} over the elements in this
     * queue.
     *
     * <p>The {@code Spliterator} reports {@link Spliterator#SIZED},
     * {@link Spliterator#SUBSIZED}, and {@link Spliterator#NONNULL}.
     * Overriding implementations should document the reporting of additional
     * characteristic values.
     *
     * @return a {@code Spliterator} over the elements in this queue
     * @since 1.8
     */
    public final Spliterator<E> spliterator() {
        return new PriorityQueueSpliterator<E>(this, 0, -1, 0);
    }

    static final class PriorityQueueSpliterator<E> implements Spliterator<E> {
        /*
         * This is very similar to ArrayList Spliterator, except for
         * extra null checks.
         */
        private final PriorityQueue<E> pq;
        private int index;            // current index, modified on advance/split
        private int fence;            // -1 until first use
        private int expectedModCount; // initialized when fence set

        /** Creates new spliterator covering the given range */
        PriorityQueueSpliterator(PriorityQueue<E> pq, int origin, int fence,
                             int expectedModCount) {
            this.pq = pq;
            this.index = origin;
            this.fence = fence;
            this.expectedModCount = expectedModCount;
        }

        private int getFence() { // initialize fence to size on first use
            int hi;
            if ((hi = fence) < 0) {
                expectedModCount = pq.modCount;
                hi = fence = pq.size;
            }
            return hi;
        }

        public PriorityQueueSpliterator<E> trySplit() {
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
            return (lo >= mid) ? null :
                new PriorityQueueSpliterator<E>(pq, lo, index = mid,
                                                expectedModCount);
        }

        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> action) {
            int i, hi, mc; // hoist accesses and checks from loop
            PriorityQueue<E> q; Object[] a;
            if (action == null)
                throw new NullPointerException();
            if ((q = pq) != null && (a = q.queue) != null) {
                if ((hi = fence) < 0) {
                    mc = q.modCount;
                    hi = q.size;
                }
                else
                    mc = expectedModCount;
                if ((i = index) >= 0 && (index = hi) <= a.length) {
                    for (E e;; ++i) {
                        if (i < hi) {
                            if ((e = (E) a[i]) == null) // must be CME
                                break;
                            action.accept(e);
                        }
                        else if (q.modCount != mc)
                            break;
                        else
                            return;
                    }
                }
            }
            throw new ConcurrentModificationException();
        }

        public boolean tryAdvance(Consumer<? super E> action) {
            if (action == null)
                throw new NullPointerException();
            int hi = getFence(), lo = index;
            if (lo >= 0 && lo < hi) {
                index = lo + 1;
                @SuppressWarnings("unchecked") E e = (E)pq.queue[lo];
                if (e == null)
                    throw new ConcurrentModificationException();
                action.accept(e);
                if (pq.modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                return true;
            }
            return false;
        }

        public long estimateSize() {
            return (long) (getFence() - index);
        }

        public int characteristics() {
            return Spliterator.SIZED | Spliterator.SUBSIZED | Spliterator.NONNULL;
        }
    }
}

```