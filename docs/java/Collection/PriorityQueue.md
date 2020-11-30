# PriorityQueue

## 属性

```java
transient Object[] queue; //集合中的容器,用于存储对象
private int size = 0; //集合中元素的数量
private final Comparator<? super E> comparator;// 集合中的元素比较器
transient int modCount = 0;//集合修改的次数
```

## 构造器

```java
public PriorityQueue() {
    // DEFAULT_INITIAL_CAPACITY = 11,数组容量
    // 比较器默认为null
    this(DEFAULT_INITIAL_CAPACITY, null);
}
//指定容器容量
public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}
//指定比较器
public PriorityQueue(Comparator<? super E> comparator) {
    this(DEFAULT_INITIAL_CAPACITY, comparator);
}
// 容量必须大于0
public PriorityQueue(int initialCapacity,
                     Comparator<? super E> comparator) {
    // Note: This restriction of at least one is not actually needed,
    // but continues for 1.5 compatibility
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
//初始化集合中的元素
public PriorityQueue(Collection<? extends E> c) {
    //如果是SortSet,指定相同的比较器,并初始化元素
    if (c instanceof SortedSet<?>) {
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
        initElementsFromCollection(ss);
    }
    //如果是PriorityQueue,同样指定比较器并初始化集合
    else if (c instanceof PriorityQueue<?>) {
        PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        initFromPriorityQueue(pq);
    }
    else {
        //比较器为null
        this.comparator = null;
        //初始化集合中的元素
        initFromCollection(c);
    }
}
```

```java
private void initElementsFromCollection(Collection<? extends E> c) {
    Object[] a = c.toArray();
    // If c.toArray incorrectly doesn't return Object[], copy it.
    //如果数组中的组件类型不是Object,拷贝元素到新数组
    if (a.getClass() != Object[].class)
        a = Arrays.copyOf(a, a.length, Object[].class);
    int len = a.length;
    //检查只有一个元素,或者比较器不为null时检查是否为空,因为空值是无法比较的
    if (len == 1 || this.comparator != null)
        for (int i = 0; i < len; i++)
            if (a[i] == null)
                throw new NullPointerException();
    this.queue = a;
    this.size = a.length;
}
```

```java
private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
    //如果是PriorityQueue的Class类,而不是继承类,将容器指向其数组引用
    if (c.getClass() == PriorityQueue.class) {
        this.queue = c.toArray();
        this.size = c.size();
    } else {
        initFromCollection(c);
    }
}
```

```java
private void initFromCollection(Collection<? extends E> c) {
    initElementsFromCollection(c);
    heapify();
}
```

```java
//二叉模型构建
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--)
        siftDown(i, (E) queue[i]);
}
```

```java
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}
```

```java
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
```

## 方法介绍

### offer

```java
//向队列中添加元素
//如果为空,抛出空指针
//判断是否需要扩大队列的容量
//如果队列为空,在第一个索引处添加该元素,否则在最后插入重排序
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);
    size = i + 1;
    if (i == 0)
        queue[0] = e;
    else
        siftUp(i, e);
    return true;
}
```

```java
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}
```

```java
private void siftUpUsingComparator(int k, E x) {
    while (k > 0) {
        //二叉堆父节点,如果父节点比字节点大,直接跳过,否则交换位置
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        if (comparator.compare(x, (E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = x;
}
```

```java
private void siftUpComparable(int k, E x) {
    //该元素需实现Comparable接口,否则会有强转异常
    //找到该父节点,比较大小,如果父节点大,直接在尾部插入该元素,如果小,则交换
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        if (key.compareTo((E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}
```

### peek

```java
//比较简单-取堆顶元素,如果队列为空返回null
@SuppressWarnings("unchecked")
public E peek() {
    return (size == 0) ? null : (E) queue[0];
}
```

### poll

```java
//获取并移除队列的第一个元素,如果队列为空返回null
@SuppressWarnings("unchecked")
public E poll() {
    if (size == 0)
        return null;
    //获取最后一个元素的索引,
    int s = --size;
    modCount++;
    E result = (E) queue[0];
    E x = (E) queue[s];
    queue[s] = null;
    //将最后一个元素添加到堆顶并调整二叉堆
    if (s != 0)
        siftDown(0, x);
    return result;
}
```

```java
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}
```

```java
//二叉堆向下调整
private void siftDownUsingComparator(int k, E x) {
    //判断是否为叶子节点,如果大于
    int half = size >>> 1;
    while (k < half) {
        int child = (k << 1) + 1;
        //左子节点
        Object c = queue[child];
        int right = child + 1;
        //如果右节点不为空,去两个子节点中的最小值
        if (right < size &&
            comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right];
        //当前元素与最小的子节点的元素比较
        if (comparator.compare(x, (E) c) <= 0)
            break;
        //如果比子节点大,切换位置
        queue[k] = c;
        k = child;
    }
    queue[k] = x;
}
```

```java
//二叉堆向下调整
@SuppressWarnings("unchecked")
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    //判断是否为叶子节点
    int half = size >>> 1;        // loop while a non-leaf
    while (k < half) {
        //与两个子节点中最小的节点比较,比其大则替换
        int child = (k << 1) + 1; // assume left child is least
        Object c = queue[child];
        int right = child + 1;
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        if (key.compareTo((E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = key;
}
```

### remove

```java
//移除队列中的指定元素
//先判断是否存在,如果不存在返回false
public boolean remove(Object o) {
    int i = indexOf(o);
    if (i == -1)
        return false;
    else {
        removeAt(i);
        return true;
    }
}
```

```java
private E removeAt(int i) {
    // assert i >= 0 && i < size;
    modCount++;
    int s = --size;
    //移除的元素如果是最后一个,置为空,help GC即可
    if (s == i) // removed last element
        queue[i] = null;
    else {
        //将队列中的最后一个元素添加到移除的索引位置,然后向下调整二叉堆
        E moved = (E) queue[s];
        queue[s] = null;
        siftDown(i, moved);
        //如果没有向下调整,则向上调整
        if (queue[i] == moved) {
            siftUp(i, moved);
            if (queue[i] != moved)
                return moved;
        }
    }
    return null;
}
```