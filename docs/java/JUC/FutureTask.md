# FutureTask

> FutureTask是多线程任务执行的基本单位,包含了任务和返回结果

## 属性介绍

```java
private volatile int state; //任务运行状态
private static final int NEW          = 0; //任务初始化状态
private static final int COMPLETING   = 1; // 正在运行
private static final int NORMAL       = 2; // 完成
private static final int EXCEPTIONAL  = 3; // 异常
private static final int CANCELLED    = 4; // 取消
private static final int INTERRUPTING = 5; // 正在中断
private static final int INTERRUPTED  = 6; // 已中断
```

```mermaid

graph LR
可能的状态演变过程

NEW --> COMPLETING --> NORMAL
NEW --> COMPLETING --> EXCEPTIONAL
NEW --> CANCELLED
NEW --> INTERRUPTING --> INTERRUPTED
```

```java
private Callable<V> callable; // 底层任务单元
private Object outcome; // 任务运行结果
/** The thread running the callable; CASed during run() */
private volatile Thread runner; //运行任务的线程
/** Treiber stack of waiting threads */
private volatile WaitNode waiters;//等待节点
```

## 方法简介

```java
//判断任务是否被取消,包括线程中断
public boolean isCancelled() {
    return state >= CANCELLED;
}
```

```java
//任务最终状态是NORMAL,EXCEPTIONAL,CANCELLED,INTERRUPTED.
//COMPLETING,INTERRUPTING属于瞬间状态
public boolean isDone() {
    return state != NEW;
}
```

```java
public void run() {
    //如果状态不是NEW,或者CAS设置runner为当前线程时失败,任务结束运行
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;// 获取执行任务
        if (c != null && state == NEW) {
            V result;
            boolean ran;//任务运行成功标识
            try {
                result = c.call();//调用任务运行结果
                ran = true;
            } catch (Throwable ex) {
                //任务执行期间发生异常,更新状态为EXCEPTIONAL,
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```