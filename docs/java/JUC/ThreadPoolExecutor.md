# ThreadPoolExecutor

## 属性介绍

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

> ctl是int类型的值,共32位,包含了两个字段的含义:
>
> * workerCount: 线程的有效数量
> * runState: 任务的运行状态
> 
> workerCount使用前29位表示,runState使用后三位表示
> 

```java
private static final int COUNT_BITS = Integer.SIZE - 3; // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;// 00011111 11111111 11111111 11111111
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS; // 运行状态,能接收新的任务并处理
private static final int SHUTDOWN   =  0 << COUNT_BITS; // 关闭 不能接收新的任务,但能处理队列中的任务
private static final int STOP       =  1 << COUNT_BITS; // 停止 不能接收新的任务,不处理队列中的任务,中断进程中的任务
private static final int TIDYING    =  2 << COUNT_BITS; // 整理 所有任务结束,worker count为0.
private static final int TERMINATED =  3 << COUNT_BITS; // 结束 运行terminated()方法之后
// RUNNING -> SHUTDOWN shutdown()方法
//(RUNNING or SHUTDOWN) -> STOP shutdownNow()方法
// SHUTDOWN -> TIDYING 当队列和池变成空的时候
// STOP -> TIDYING 当池为空的时候
// TIDYING -> TERMINATED terminated()
```

```java
private final BlockingQueue<Runnable> workQueue; //任务队列
```

```java
private int largestPoolSize;//线程池最大值
```

```java
private long completedTaskCount;//完成的任务数量
```

```java
private volatile ThreadFactory threadFactory;//线程工厂,用于生产并获取线程对象
```

```java
//当线程空闲时等待工作的超时时间(以纳秒为单位),这个值只有在超过corePoolSize的时候或者allowCoreThreadTimeOut=true才有意义
private volatile long keepAliveTime;
```

```java
//如果为false,空闲的线程保持激活状态,如果为true,空闲的线程将等待keepAliveTime的时间
private volatile boolean allowCoreThreadTimeOut;
```

```java
// 激活的worker的最小值,如果allowCoreThreadTimeOut设置为true,此值为0
private volatile int corePoolSize;
```

```java
//线程池中的最大值,不超过 1<<29
private volatile int maximumPoolSize;
```

## 方法

### execute

```java
//运行任务
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * 运行任务三个步骤:
     * 1.如果正在运行的线程少于corePoolSize,尝试创建一个新的线程开启这个任务,作为worker的第一个命令.调用addWork方法检查
     * 		runState和workerCount,防止添加不应该添加的线程
     * 2. 如果大于核心线程数或者addWorker方法返回false(说明情况会返回false),重新判断线程池的运行状态,如果是RUNNING状态,并将
     *		任务添加到阻塞队列成功,再次检查线程池运行状态 如果是非RUNNING状态并回滚队列,调用拒绝策略拒绝执行(只有RUNNING状态才	   *      接收新任务).如果当前线程任务数为0,创建非核心的空线程
     * 3.上述情况均不满足时,添加一个非核心线程启动任务,失败时执行拒绝策略
     */
    //获取ctl
    int c = ctl.get();
    //如果work的数量小于corePoolSize,将Runnable添加到队列中, 调用addWorker方法.
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

```java
//线程启动成功返回true,否则返回false
// core 为true 以corePoolSize为上限,否则以maximumPoolSize为上线
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        //获取运行状态
        int rs = runStateOf(c);

        //  检查runstate,队列不为空.因为SHUTDOWN状态需要允许完队列中的任务,线程池才能结束,也就是说rs== RUNNING或者(rs==SHUTDOWN 并且firstTask == null 并且 队列不为空,继续执行)
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            //查询worker的数量
            int wc = workerCountOf(c);
            //如果超出最大值返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // cas 设置workcounter += 1,结束循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            //r如果状态被其他线程改变了,重新自旋设置
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false; //worker启动成功标识
    boolean workerAdded = false; // worker添加成功标识
    Worker w = null;
    try {
        //创建一个worker对象
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            //同步锁,用户同步任务调度,状态的修改
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
				//如果是Running状态或者SHUTDOWN并且任务为空,将其添加到集合中,(SHUTDOWN状态下,仍需要执行队列中的任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    //记录worker的最大数
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;//标记worker添加成功
                }
            } finally {
                mainLock.unlock();
            }
            //worker添加成功,线程启动(启动后执行了那些)
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        //如果worker启动失败,在workers集合中移除worker,workercount - 1,终结空闲的线程
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

```java
//worker启动失败时执行此方法.
//1.从workers中尝试移除
//2.worker count - 1
//3.尝试结束
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        decrementWorkerCount();
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

```java
//尝试终结线程池
//如果线程是RUNNING状态,或者SHUTDOWN状态并且阻塞队列为空,或者已经是TIDYING状态或者TERMINATED不允许终结
//设置运行状态TIDYING,执行terminated()方法,设置运行状态TERMINATED
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        if (workerCountOf(c) != 0) { // Eligible to terminate
            //中断一个空闲的线程
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

```java
//线程启动后执行的方法
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //如果worker中有任务,执行worker中的任务,否则去阻塞队列中获取任务
        while (task != null || (task = getTask()) != null) {
            //加锁,一个work只能执行一个任务
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            //如果线程池是STOP状态,中断线程,不再运行任务
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 执行完成任务后推出
        processWorkerExit(w, completedAbruptly);
    }
}
```

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    //如果意外退出,workcount递减
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();
	//移除线程
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

```java
//获取阻塞队列中的任务
//以下条件会返回空:
//1.workers的数量超过maximumPoolSize
//2.运行状态为STOP
//3.运行状态shutdown并且阻塞队列为空
//4.超时等待
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        //由于超过keepAliveTime的线程会被销毁,所以需在指定时间内获取任务
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

### shutdown

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();//检查是否有修改线程的权限
        advanceRunState(SHUTDOWN); //将线程设置为SHUTDOWN状态
        interruptIdleWorkers();//中断空闲的线程
        onShutdown(); // hook for ScheduledThreadPoolExecutor 钩子函数
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

### shutdownNow

```java
//立刻停止任务运行,并返回阻塞队列中的任务
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);//设置运行状态STOP
        interruptWorkers(); // 中断所有线程中执行的任务
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

