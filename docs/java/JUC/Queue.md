# Queue<E>队列接口

```java
boolean add(E e);
```

> 将元素插入到队列,如果成功返回true,否则抛出IllegalStateException

```java
boolean offer(E e);
```

> 与add方法不同的是插入队列失败时返回false

```java
E remove();
```

> 移除队列的第一个元素,返回移除的元素,如果队列是空的抛出NoSuchElementException

```java
E poll();
```

> 获取并移除队列的第一个元素,如果队列为空返回null

```java
E element();
```

> 获取队列的第一个元素,如果队列为空,抛出NoSuchElementException

```java
E peek();
```

> 获取队列的第一个元素,如果队列为空返回null