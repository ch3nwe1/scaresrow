# StampedLock(不可重入乐观读写锁)

## Constructor

```java
/** * Creates a new lock, initially in unlocked state. */
//ORIGIN = WBIT << 1 = 256 = 1 0000 0000
//初始化锁的状态值为256
public StampedLock() {  state = ORIGIN;}
```

## Method

### writeLock

```java
//独占式获取锁
public long writeLock() {
    long s, next;  // bypass acquireWrite in fully unlocked case only
    // ABITS = 255 = 1111 1111
    //1 0000 0000 & 1111 1111 = 0
    // 
    return ((((s = state) & ABITS) == 0L &&
             U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
            next : acquireWrite(false, 0L));
}
```

