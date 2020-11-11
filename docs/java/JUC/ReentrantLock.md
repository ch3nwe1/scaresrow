# ReentrantLock(可重入同步锁)

​	ReentrantLock实现了Lock接口

```java
//加锁
void lock();
```

```java
//可中断的锁
void lockInterruptibly()
```

```java
//尝试加锁,成功返回true,否则返回false
boolean tryLock();
```

```java
//在指定时间内获取锁,如果获取成功返回true,否则返回false,如果线程被中断则抛出InterruptedException
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
```

```java
//释放锁
void unlock();
```

```java
//返回 Condition实例,前提是当前线程必须拥有锁,用户线程等待或者唤醒的操作
Condition newCondition();
```

ReentrantLock同步锁中包含了公平锁和非公平锁,其底层是基于AQS实现的,我们分别研究下这两种锁是如何实现的.如何创建公平锁与非公平锁, 可以通过其构造器得知.

```java
//默认是非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}
```

```java
//true为公平锁,false为非公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

## 公平锁

### lock(加锁)

```java
public void lock() {
    //同步对象加锁
    sync.lock();
}
```

```java
final void lock() {
    //获取一把锁
    acquire(1);
}
```

```java
public final void acquire(int arg) {
    // tryAcquire方法尝试获取一把锁,如果返回true代表获取锁成功,如果获取失败,将当前线程添加到等待队列,并标	 // 记独占模式,如果acquireQueued返回true代表线程需要中断
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //如果线程中断状态为true,调用线程中断方法,原因是获取中断状态后,线程状态会被清理
        selfInterrupt();
}
```

```java
//尝试获取锁
protected final boolean tryAcquire(int acquires) {
    //获取当前竞争锁的线程
    final Thread current = Thread.currentThread();
    //锁的状态,如果为0,代表处于空闲状态,大于0,其值代表锁的处理,在使用锁的阶段,加锁和释放锁都需要成对出现,
    //即调用lock()后需要调用unlock()来释放,每次加锁state+1,释放锁时state-1
    int c = getState();
    //如果锁状态为0,代表空闲
    if (c == 0) {
        //hasQueuedPredecessors()方法判断同步队列中是否有排在当前线程之前的线程竞争锁,这是公平锁的体现.
        //同步队列是使用FIFO的设计,即先进先出
        //compareAndSetState()方法使用CAS操作设置state值,返回true代表设置成功,这里为什么使用CAS操作,个人认为是一种悲观的体现,当前线程始终认为有其他线程在抢锁,所以自身也尝试竞争是否能够成功
        //setExclusiveOwnerThread()方法设置锁被独占的线程,最终返回true代表取锁成功
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //判断当前线程是否与独占锁的线程是否相等,如果是同一个线程,允许获取锁,因为这是一个可重入的锁
    else if (current == getExclusiveOwnerThread()) {
        //加锁的次数
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        //设置state,这里为什么不使用CAS操作?因为当前线程已经持有锁,不会有其他线程竞争
        setState(nextc);
        return true;
    }
    return false;
}
```

```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    //判断当前线程在队列之前是否还有竞争者在排队,FIFO队列,首先的条件是head != tail 因为head和tail相等时队列是空的,其次判断head的next节点的线程不等于当前线程,代表当前线程不是排在第一个
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

```java
//前面讲到如果tryAcquire(int)方法获取失败,会将当前线程添加到队列中等待,我们来看一看是如何添加到队列中的?
final boolean acquireQueued(final Node node, int arg) {
    //failed 属性是一个失败的标识,如果获取锁失败,取消当前线程去竞争锁,如果取消稍后解释
    boolean failed = true;
    try {
        //线程中断标识,如果为true,代表线程需要中断,在这个方法返回后调用
        boolean interrupted = false;
        //这里是自旋
        for (;;) {
            //获取当前线程的节点的前一个节点
            final Node p = node.predecessor();
            //如果前一个节点是头节点,则尝试获取锁,获取锁的结果决定当前线程的走向
            //1.获取锁成功,将当前节点设置为头节点,并返回
            if (p == head && tryAcquire(arg)) {
                // head = node; node.thread = null; node.prev = null;
                setHead(node);
                p.next = null; // help GC 原来的head节点的next设置为null,取消与node的引用关系
                failed = false;// 取锁成功,failed设置为false
                return interrupted;//返回interrupted中断标识
            }
            //2.如果获取锁失败,判断当前是否应该挂起,如果应该挂起,则执行挂起,并判断线程是否中断
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        //如果失败,取消锁的竞争
        if (failed)
            cancelAcquire(node);
    }
}
```

```java
//添加一个节点到等待队列, mode包含独占模式和共享模式
private Node addWaiter(Node mode) {
    //以指定模式创建当前的节点
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    //尝试将当前节点添加到队列末尾
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            //如果添加成功,返回当前节点
            return node;
        }
    }
    // compareAndSetTail() CAS操作设置tail节点失败后,通过enq()方法添加到tail节点,该方法是以自旋的方式直到添加成功为止
    enq(node);
    return node;
}
```

```java
//将当前节点添加到队列的末尾
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            //如果tail节点未初始化,需要创建一个空节点来初始化队列
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

```java
//判断node节点是否应该挂起
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //前一个节点的等待状态
    int ws = pred.waitStatus;
    //如果是SIGNAL状态,返回true
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        //ws > 0 代表该节点已经取消,向前查找未取消的节点,将队列拼接上,在这里有一个疑问,取消的节点已经离开队列,是否应该help GC一下呢?
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        //这里更新前一个节点的状态为SIGNAL,而没有返回true来挂起线程,为了重试获取锁确保当前线程没有成功才执行挂起
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

```java
private final boolean parkAndCheckInterrupt() {
    //线程挂起
    LockSupport.park(this);
    //线程是否中断
    return Thread.interrupted();
}
```

```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    //如果节点为null,直接返回
    if (node == null)
        return;

    node.thread = null;//将当前节点的线程设置为null

    // Skip cancelled predecessors
    //获取当前节点的前一个节点,如果前一节点状态为是取消的,继续向前查找
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    //前一个节点的下一个节点
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    //标记当前节点的状态
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    //如果当前节点是tail节点,则更新前一节点为tail,并将前一个节点的next设置为null
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        //1.前一节点不是head节点
        //2.前一个节点的状态SIGNAL或者状态小于等于0然后CAS更新为SIGNAL成功
        //3.前一个节点的线程不是空
        //以上条件满足,获取当前节点的下一个节点,此节点不是空并且状态小于等于0,将前一个节点与下一个节点拼起来
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            //否则唤醒当前节点的下一个节点?
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

### lockInterruptibly(可中断加锁)

```java
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
```

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    //如果线程中断,抛出中断异常
    if (Thread.interrupted())
        throw new InterruptedException();
    //尝试获取锁.如果失败,执行doAcquireInterruptibly()
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    //首先添加一个独占模式的线程节点到队列尾部
    final Node node = addWaiter(Node.EXCLUSIVE);
    //下面这些实现与acquireQueued()方法基本相同,唯一的区别是如果挂起的线程被唤醒后如果线程是中断状态则抛出中断异常
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### tryLock(尝试加锁)

​	tryLock是不公平锁的一种体现,它获取锁的时候不会排队,在写到不公平锁的时候详细解释一下

```java
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}
```

```java
final boolean nonfairTryAcquire(int acquires) {
    //获取当前线程
    final Thread current = Thread.currentThread();
    int c = getState();
    //如果锁处于空闲,直接获取锁,并这是独占锁的线程为当前线程
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //如果不是空闲,判断持有锁的线程是当前线程,添加锁的次数,获取锁成功,因为这是可重入锁
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    //其他情况都表示获取锁失败
    return false;
}
```

### tryLock(指定时间内尝试获取锁)

```java
public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    //将事件转换为纳秒
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    //如果线程是中断状态,直接抛出中断异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 首先尝试获取锁,如果失败调用doAcquireNanos()
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    //超时时间如果小于等于0,直接获取锁失败
    if (nanosTimeout <= 0L)
        return false;
    //结束时间
    final long deadline = System.nanoTime() + nanosTimeout;
    //添加一个独占的节点到队列的末尾
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        //自旋
        for (;;) {
            //获取当前节点的前一个节点
            final Node p = node.predecessor();
            //如果前一个节点是head节点,尝试获取锁,成功返回true
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            //超时的结束时间与当前时间差,如果大于0代表还没有超时,继续自旋
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            //判断是否应该挂起线程,如果时间差大于1000L,将其挂起直到超时时间结束,再自旋开始最后获取一次锁
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        //如果获取锁失败了,取消当前线程在队列中的节点
        if (failed)
            cancelAcquire(node);
    }
}
```

### unlock(释放锁)

```java
public void unlock() {
    sync.release(1);
}
```

```java
public final boolean release(int arg) {
    //尝试释放锁,如果成功,唤醒下一个节点
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

```java
protected final boolean tryRelease(int releases) {
    //将锁的state释放,即减去release的值
    int c = getState() - releases;
    //释放锁的前提是当前线程必须拥有锁
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    //空闲标识
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

## 不公平锁

### lock(加锁)

```java
//首先直接去获取锁,这里没有尝试,如果失败再尝试获取通过调用tryAcquire
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

```java
protected final boolean tryAcquire(int acquires) {
    //前文已经解释过了,nonfairTryAcquire()方法如果锁空闲,直接获取锁,而公平锁则是查看前面是否有排队的线程
    return nonfairTryAcquire(acquires);
}
```

## Condition(线程的等待与唤醒)

```java
//线程等待,可中断
void await() throws InterruptedException;
```

```java
//不可中断的等待
void awaitUninterruptibly();
```

```java
long awaitNanos(long nanosTimeout) throws InterruptedException;
```

```java
boolean await(long time, TimeUnit unit) throws InterruptedException;
```

```java
boolean awaitUntil(Date deadline) throws InterruptedException;
```

```java
void signal();
```

```java
void signalAll();
```

### await

```java
public final void await() throws InterruptedException {
    // 首先线程如果是中断状态, 抛出InterruptedException异常
    if (Thread.interrupted())
        throw new InterruptedException();
    //添加到等待队列
    Node node = addConditionWaiter();
    //释放当前线程持有的锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

```java
private Node addConditionWaiter() {
    //获取最后一个等待者
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    //如果最后一个等待线程不是null并且节点的状态不是CONDITION,清理已经取消的等待线程
    if (t != null && t.waitStatus != Node.CONDITION) {
        //删除非Condition的节点
        unlinkCancelledWaiters();
        //重新赋值t
        t = lastWaiter;
    }
    //创建一个Condition状态的节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    //如果队列为空firstWaiter = lastWaiter = node,否则t.nextWaiter = node
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

```java
//清理已经取消的等待线程
private void unlinkCancelledWaiters() {
    //队列第一个节点
    Node t = firstWaiter;
    //遍历过的最后一个非Condition节点,也可以说是当前遍历节点的前一个节点,因为Condition Queue是一个单向链表
    Node trail = null;
    while (t != null) {
        //下一个等待节点
        Node next = t.nextWaiter;
        //节点的等待状态不是CONDITION
        if (t.waitStatus != Node.CONDITION) {
            //断开当前节点与下一节点的链接
            t.nextWaiter = null;
            //如果前一个节点为null,更新next为第一个等待队列,否则next节点为trail的nextWaiter
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            //如果next为null,代表已经到了最后,更新lastWaiter为trail,因为t节点的状态不是CONDITION
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```

```java
//线程等待期间将线程持有的重入锁全部释放, 如果失败将节点的状态waitStatus设置为CANCELLED
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```