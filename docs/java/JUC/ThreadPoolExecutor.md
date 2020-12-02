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
     * 2.如果将任务成功的添加到队列中,我们还需要确定是否应该添加一个新的线程,
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
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        //获取运行状态
        int rs = runStateOf(c);

        //  检查runstate,队列不为空
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
            // cas 设置workcounter += 1
            if (compareAndIncrementWorkerCount(c))
                break retry;
            //r如果状态被其他线程改变了,重新自旋设置
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
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
				//如果是Running状态或者SHUTDOWN并且任务为空,将其添加到集合中
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
            //worker添加成功,线程启动
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
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        if (workerCountOf(c) != 0) { // Eligible to terminate
            //中断线程空闲的
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
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
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
        processWorkerExit(w, completedAbruptly);
    }
}
```