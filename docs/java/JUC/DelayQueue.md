# DelayQueue

> DelayQueue是延迟队列，元素只有在指定时间后才可以取出，队列中的元素需要实现Delayed接口，返回剩余等待的时间

```java
// 添加元素到队列中
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        q.offer(e);
        //线程唤醒
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }	
        return true;
    } finally {
        lock.unlock();
    }
}
```

```java
// 非阻塞式获取并移除队列中的第一个元素 
public E poll() {
     final ReentrantLock lock = this.lock;
     lock.lock();
     try {
         E first = q.peek();
         // 如果队列为空或者时间还为到，返回null
         if (first == null || first.getDelay(NANOSECONDS) > 0)
             return null;
         else
             return q.poll();
     } finally {
         lock.unlock();
     }
 }
```

```java
//阻塞式获取并移除队列的第一个元素
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        //自旋
        for (;;) {
            //如果队列为空，线程等待
            E first = q.peek();
            if (first == null)
                available.await();
            else {
                //如果队列的一个元素不为空，获取延迟时间，如果小于等于0，直接移除并返回
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return q.poll();
                first = null; // don't retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    //更新leader线程
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```

```java
//阻塞式，指定之时间内获取并移除队列中的元素
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null) {
                if (nanos <= 0)
                    return null;
                else
                    nanos = available.awaitNanos(nanos);
            } else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return q.poll();
                if (nanos <= 0)
                    return null;
                first = null; // don't retain ref while waiting
                if (nanos < delay || leader != null)
                    nanos = available.awaitNanos(nanos);
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        long timeLeft = available.awaitNanos(delay);
                        nanos -= delay - timeLeft;
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```

```java
//获取队列的第一个元素
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return q.peek();
    } finally {
        lock.unlock();
    }
}
```

