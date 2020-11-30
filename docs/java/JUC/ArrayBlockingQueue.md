# ArrayBlockingQueue
> ArrayBlockingQueue是基于数组实现的阻塞队列，队列中的元素排列按照先进先出的顺序。head再队列中时间最长，tail在队列中时间最短。新元素被插入到队列的尾部（tail）。获取元素从队列的头部（head）。
>
> 数组中的元素固定，一旦创建不能被改变，如果将元素放入已满的队列将会阻塞，直到有元素被提取。同样提取一个空队列的元素将会阻塞，直到有元素插入队列。这个类支持公平与非公平两种策略，公平策略会降低吞吐量但也会减少饥饿

## 属性介绍

- final Object[] items; 用于存放队列中的元素，final代表一旦创建不可变
- int takeIndex; 队列中获取和移除元素的索引
- int putIndex; 队列中添加元素的索引
- int count;队列中元素的数量
- final ReentrantLock lock; 排他锁，用于队列读写同步
- private final Condition notEmpty;用于队列不为空时线程唤醒的条件
- private final Condition notFull;用于队列不满时线程唤醒的条件
- transient Itrs itrs = null; ？？？

## 构造器

```java
//capacity 队列的长度，默认非公平策略
public ArrayBlockingQueue(int capacity) {    this(capacity, false);}
```

```java
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    //初始化队列数组
    this.items = new Object[capacity];
    //初始化同步锁
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

```java
//向队列中添加默认元素
public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
    this(capacity, fair);

    final ReentrantLock lock = this.lock;
    //开始我还在怀疑，即使多线程环境，也没有必要在构造器中加锁，所以在这里加锁只是为了增加共享变量的可见性
    lock.lock(); // Lock only for visibility, not mutual exclusion
    try {
        int i = 0;
        try {
            for (E e : c) {
                //检查是否为空
                checkNotNull(e);
                items[i++] = e;
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();//如果集合c中元素的数量大于队列的长度
        }
        count = i;//更新元素的数量
        putIndex = (i == capacity) ? 0 : i;//更新tail的位置，如果到了最后，再从队列的第一个开始
    } finally {
        lock.unlock();
    }
}
```

## 方法实现

### offer

```java
public boolean offer(E e) {
    //	检查是否为空
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    //同步锁
    lock.lock();
    try {
        //如果数量已满，返回false
        if (count == items.length)
            return false;
        else {
            //添加到队列中
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```

```java
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    // 将元素添加到队列中，如果到了容器的最后位置，再从头开始，元素的数量+1，并且队列不为空的条件唤醒
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    notEmpty.signal();
}
```

### put

```java
// put方法属于阻塞式添加元素，直到添加成功为止
public void put(E e) throws InterruptedException {
    //检查元素是否为空
    checkNotNull(e);
    //胡哦去同步锁
    final ReentrantLock lock = this.lock;
    //可中断的加锁，如果线程中断，抛出异常
    lock.lockInterruptibly();
    try {
        //如果队列已满，线程等待，否则将其添加到队列中
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```

### off(E e, long timeout, TimeUnit unit)

```java
// 在指定时间内添加元素到队列，如果成功返回true,可能会阻塞
public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
	//检查是否为空
    checkNotNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(e);
        return true;
    } finally {
        lock.unlock();
    }
}
```

### poll

```java
// 获取并移除队列head,非阻塞式，队列不为空返回head，为空返回null
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //dequeue移除队列
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
```

```java
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];//取出head位置的元素
    items[takeIndex] = null;//置为空。
    if (++takeIndex == items.length)//头位置+1,如果到了容器最后的索引，返回到开始位置
        takeIndex = 0;
    count--;//元素的数量-1
    if (itrs != null)
        itrs.elementDequeued(); // ？？？稍后解析
    notFull.signal(); //容器不满的条件唤醒
    return x;//返回该元素
}
```

### take

```java
//阻塞式获取并移除队列的第一个元素，直到获取成功为止，如果线程中断，抛出异常
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

### poll(long timeout, TimeUnit unit)

```java
//指定时间内获取并移除队列中的head元素
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

### peek

```java
//获取队列中的head元素,如果队列为空返回null
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex); // null when queue is empty
    } finally {
        lock.unlock();
    }
}
```

### count

```java
//返回队列中元素的数量
public int size() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return count;
    } finally {
        lock.unlock();
    }
}
```

### remainingCapacity

```java
//返回队列中的剩余容量
public int remainingCapacity() {
     final ReentrantLock lock = this.lock;
     lock.lock();
     try {
         return items.length - count;
     } finally {
         lock.unlock();
     }
}
```

### remove

```java
//移除队列中的指定元素
//如果对象为空，获取队列为空，或者元素不存在于队列中，返回false
//从head开始向tail查找
public boolean remove(Object o) {
    if (o == null) return false;
    final Object[] items = this.items;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count > 0) {
            final int putIndex = this.putIndex;
            int i = takeIndex;
            do {
                if (o.equals(items[i])) {
                    removeAt(i);
                    return true;	
                }
                if (++i == items.length)
                    i = 0;
            } while (i != putIndex);
        }
        return false;
    } finally {
        lock.unlock();
    }
}
```

```java
//移除指定位置的元素
void removeAt(final int removeIndex) {
    // assert lock.getHoldCount() == 1;
    // assert items[removeIndex] != null;
    // assert removeIndex >= 0 && removeIndex < items.length;
    final Object[] items = this.items;
    //如果移除的是head元素，将该索引位置的元素置为空，head向后移动，元素数量-1
    if (removeIndex == takeIndex) {
        // removing front item; just advance
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    } else {
        // an "interior" remove

        // slide over all others up through putIndex.
        final int putIndex = this.putIndex;
        for (int i = removeIndex;;) {
            int next = i + 1;
            if (next == items.length)
                next = 0;
            if (next != putIndex) {
                items[i] = items[next];
                i = next;
            } else {
                items[i] = null;
                this.putIndex = i;
                break;
            }
        }
        count--;
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    notFull.signal();
}
```

