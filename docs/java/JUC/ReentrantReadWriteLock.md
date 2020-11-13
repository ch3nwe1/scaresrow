# ReentrantReadWriteLock(可重入读写锁)

> ReentrantLock是独占式可重入排它锁,也就是说只有一个线程持有锁,其他线程需要等待持锁线程释放后才能获得,但是在实际场景中,如果多个线程访问共享资源时只是读取的,这实际上是线程安全的,没有必要等待线程释放锁.所以,此类设计实为读写分离的设计.如果锁属于读锁,那其他线程是可以获取的.属于共享锁的性质.同样也是基于AQS实现其ReadWriteLock接口功能的锁

## ReadWriteLock API

```java
//获取读锁
Lock readLock();
```

```java
//获取写锁
Lock writeLock();
```

> 这两个API的实现亦是简单关键在于其接口返回的lock对象是如何实现的,让我们一步一步接口他的面纱

```java
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

## ReadLock

### lock

```java
public void lock() {
    sync.acquireShared(1);
}
```

```java
public final void acquireShared(int arg) {
    //首先尝试获取共享锁,返回值为1 代表成功 -1 代表失败
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

```java
protected final int tryAcquireShared(int unused) {
    //获取当前的线程
    Thread current = Thread.currentThread();
    int c = getState();
    //首先判断是否有写入锁, 如果有写入锁并且不是当前的线程持有写入锁,直接返回-1,获取失败
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    //获取读锁的数量
    int r = sharedCount(c);
    //readerShouldBlock方法判断是否应该阻塞,该方法实现取决于公平锁与非公平锁的实现而异,如果是公平锁,需要
    //查找队列中的第一个线程是不是自身的线程,非公平锁需要判断同步队列的第一个线程是否独占模式的线程,如果是独	  //占线程,返回true,获取锁应该阻塞,否则返回false
    if (!readerShouldBlock() &&
        r < MAX_COUNT && // 锁的数量小于65535
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            //如果是第一次读取,更新firstReader为当前线程,记录第一次获取读锁的线程和数量
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            //如果不为0,并且当前线程为第一次获取读锁的线程,更新读锁的数量
            firstReaderHoldCount++;
        } else {
            //如果锁的数量不为0,并且不是第一个读锁的线程,需要缓存当前线程持有锁的数量
            HoldCounter rh = cachedHoldCounter;//最后一个读锁线程持有的数量
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}

final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        // 如果存在独占锁,并且不为当前线程,直接返回-1
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) {
            //如果读锁应该阻塞,并且当前线程不是第一个读锁的线程,当前线程持有锁的数量等于0,返回-1.失败
            //这里为什么当前线程持有锁的数量为0,为什么宣布失败呢?
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        //如果共享锁超过了最大数量,抛出异常
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        //CAS设置state成功,则获取锁成功
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

```java
//EXCLUSIVE_MASK (1 << SHARED_SHIFT) - 1 = 65535 = 0x7fff
//state是一个int类型的值,共32位,前16位代表写入锁的数量,后16位代表读锁的数量
//readLock负责后面16位的改变,writeLock负责前16位的值
//这里是c & 0x7fff 明显是取前16位的值,以此来判断是否有写入锁
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
//state无符号右移16位,取读锁的数量
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
```

```java
//公平锁读取时是否应该阻塞,取决于队列之前是否有线程等待,究竟何时会有线程添加到队列中?先不着急,后面肯定会有
final boolean readerShouldBlock() {
    return hasQueuedPredecessors();
}
```

```java
//非公平锁判断线程是否应该阻塞
final boolean readerShouldBlock() {
    return apparentlyFirstQueuedIsExclusive();
}
//判断队列中的第一个线程是否独占
final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
```

```java
//当尝试获取共享锁失败时,添加到等待队列,自旋获取锁
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //如果前一个节点是头节点,尝试获取共享锁
            if (p == head) {
                int r = tryAcquireShared(arg);
                //设置共享锁成功后
                if (r >= 0) {
                    //当前节点设置为头节点,并传播
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```