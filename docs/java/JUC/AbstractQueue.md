# AbstractQueue

> AbstractQueue是抽象的队列实现

```java
// 向队列中添加元素，属于非阻塞实现，如果成功返回true,如果失败抛出异常
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```

```java
//获取移除队列的元素，与poll的区别在于此方法移除失败时会抛出异常，同样属于非阻塞实现
public E remove() {
    E x = poll();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
```

```java
//获取队列中的第一个元素
//与peek()方法的区别：
//1.非阻塞并且队列不为空时返回队列的head
//2.如果队列为空：element方法将会抛出异常。而peek()方法返回null
public E element() {
    E x = peek();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
```

```java
//清空队列
public void clear() {
    while (poll() != null)
        ;
}
```

```java
//将一个集合中的元素添加到队列中，如果添加过返回true,否则返回false
public boolean addAll(Collection<? extends E> c) {
     if (c == null)
         throw new NullPointerException();
     if (c == this)
         throw new IllegalArgumentException();
     boolean modified = false;
     for (E e : c)
         if (add(e))
             modified = true;
     return modified;
 }
```

