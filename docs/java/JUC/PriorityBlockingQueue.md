# PriorityBlockingQueue

> 阻塞的优先级队列,基于二叉堆实现

```java
// 添加一个元素到指定队列
// 如果元素为null,抛出空指针异常
// add put offer(E e, long timeout, TimeUnit unit) 非阻塞
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    //检查是否需要扩容
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        // 二叉堆向上调整
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        //不为空的条件唤醒
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}
```

```java
//获取并移除第一个元素,如果队列为空,返回null 非阻塞
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

```java
// 获取并移除队列的第一个元素,如果队列为空,发生阻塞
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}
```

```java
// 阻塞式在指定时间内获取并移除队列的第一个元素
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null && nanos > 0)
            nanos = notEmpty.awaitNanos(nanos);
    } finally {
        lock.unlock();
    }
    return result;
}
```

```java
//获取队列的第一个元素,如果为空返回null
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (size == 0) ? null : (E) queue[0];
    } finally {
        lock.unlock();
    }
}
```

```java
//移除指定元素,如果不存在返回false
public boolean remove(Object o) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = indexOf(o);
        if (i == -1)
            return false;
        removeAt(i);
        return true;
    } finally {
        lock.unlock();
    }
}
```

```java
//将队列中的元素移除,并添加到指定的集合中
public int drainTo(Collection<? super E> c, int maxElements) {
    if (c == null)
        throw new NullPointerException();
    if (c == this)
        throw new IllegalArgumentException();
    if (maxElements <= 0)
        return 0;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int n = Math.min(size, maxElements);
        for (int i = 0; i < n; i++) {
            c.add((E) queue[0]); // In this order, in case add() throws.
            dequeue();
        }
        return n;
    } finally {
        lock.unlock();
    }
}
```