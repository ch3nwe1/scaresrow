# BlockingQueue<E>

```java
boolean add(E e);
```

> 添加指定元素到队列中,如果队列已满,抛出IllegalStateException

```java
boolean offer(E e);
```

> 添加指定元素到队列中,如果队列已满,返回false

```java
void put(E e) throws InterruptedException;
```

> 插入指定元素到队列中,如果队列已满,线程等待直到队列中有其他元素被移除,如果等待过程中线程中断抛出异常

```java
boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException;
```

> 在指定时间内插入元素,成功返回true,超时后返回false,等待期间线程中断抛出异常

```java
E take() throws InterruptedException;
```

> 获取并移除队列的第一个元素,线程等待直到队列中的元素可以移除,如果线程中断抛出异常

```java
E poll(long timeout, TimeUnit unit)
    throws InterruptedException;
```

> 在指定时间内等待移除并获取队列的第一个元素,超时将返回null,等待期间线程中断将抛出异常

```java
int remainingCapacity();
```

> 返回队列中的剩余容量

```java
boolean remove(Object o);
```

> 移除队列中指定的元素,如果成功返回true,否则返回false

```java
public boolean contains(Object o);
```

> 如果队列中包含指定元素返回true,否则返回false

```java
int drainTo(Collection<? super E> c);
```

> 将队列中所有的元素移除,添加到集合c中,返回传输的数量

```java
int drainTo(Collection<? super E> c, int maxElements);
```

> 将队列中所有的元素移除,添加到集合c中,最多添加maxElements的数量,返回实际传输的数量