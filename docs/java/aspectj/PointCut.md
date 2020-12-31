# 切点

> 横切方法

```java
call(void Point.setX(int))
```

> 切多个方法

```java
call(void Point.setX(int)) || call(void Point.setY(int))
```

> 多个类的多个方法

```java
pointcut move():
    call(void FigureElement.setXY(int,int)) ||
    call(void Point.setX(int))              ||
    call(void Point.setY(int))              ||
    call(void Line.setP1(Point))            ||
    call(void Line.setP2(Point));
```