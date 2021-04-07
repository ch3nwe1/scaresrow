# Redis客户端程序设计：

本应用为redis 图形化视图程序，通过连接服务端，提供redis各种操作API.

## 1. 新建连接

用户需提供host,port,password信息连接到redis服务端，此为一个会话。该程序可建立多个会话，也就是说连接多个redis服务端，每个会话可打开一个窗口，进行redis操作。同时用户信息可作持久化保存到本地文件中，用户可不必每次都输入信息进行连接

## 2. redis操作API

redis共有五种数据结构，包括

1. 字符串Strings
2. hashes 
3. 链表 	lists
4. 不重复的无序的集合 sets
5. 有序集合 sorted sets

###  del 

> 格式：del key [key...]

> 说明：删除一个或者多个key

> 返回：删除的key的数量

### dump

>  格式：dump key

> 说明： 序列化key
>
> 返回： 序列化后的key

### exists

> 格式：exists key [key...]
>
> 说明：判断key是否存在
>
> 返回：如果只有一个key的时候存在返回1，不存在返回0，如果多个key，返回存在的key的个数

### copy

> 格式：copy sourceKey tagetKey
>
> 说明： 复制一个sourceKey 的值作为另一个tagetKey的值
>
> 返回： 1 表示复制成功，0表示复制失败

```shell
SET dolly "sheep"
COPY dolly clone
GET clone
```

### expire 

> 格式： EXPIRE key seconds
>
> 说明：为key设置生命周期，如果超时将会自动删除key
>
> 返回：1返回设置成功，0表示key不存在或者不能设置过期时间

### expireat

> 格式：EXPIREAT key timestamp
>
> 说明：为key设置生命周期，与expire不同的是使用的unix时间戳
>
> 返回：1成功0失败

### keys

> 格式： KEYS pattern
>
> 说明： 查询符合给定模式pattern的key，支持正则表达式
>
> 返回：一个满足条件的集合

### move

> 格式：move key db
>
> 说明：移动指定的key到目标数据库
>
> 返回：1迁移成功0失败

### presist

> 格式：presist key
>
> 说明：持久化key
>
> 返回：1成功0失败

### pexpire

> 格式：PEXPIRE key milliseconds
>
> 说明：设置key的过期时间，以毫秒为单位
>
> 返回：1成功0失败

### pexpireat

> 格式：PEXPIREAT key milliseconds-timestamp
>
> 说明：与pexpire功能类似，以毫秒为单位的unix时间戳
>
> 返回：1成功0失败

### ttl

> 格式：ttl key
>
> 说明：查询key的过期时间，以秒为单位
>
> 返回：key的剩余过期时间，如果key不存在返回-2，如果key存在并且没有设置过期时间或者持久化返回-1

### pttl

> 与ttl相似，不同的是该命令以毫秒为单位

### randomkey

> 格式: randomkey
>
> 说明：随机返回key
>
> 返回：随机的key，如果数据库中没有key返回nil

### rename

> 格式：RENAME key newkey
>
> 说明：将key重命名为newkey
>
> 返回: 如果key与newkey名字相同返回错误，如果newkey已经存在，值将会被覆盖

### renamenx

> 格式：renamenx key newkey
>
> 说明：当且仅当 newkey 不存在时，将 key 改名为 newkey 。当 key 不存在时，返回一个错误。

### restore

> 反序列化给定值 ？？？ 待研究

### type

> 格式： type key
>
> 说明： 查询key的类型
>
> 返回：string,hash,list,set,zset

### UNLINK

> 与del相似，区别在于异步删除，不存在的key跳过，非阻塞

### touch

> 格式：TOUCH key [key ...]
>
> 说明：修改指定key(s) 最后访问时间 若key不存在，不做操作
>
> 返回：操作的key的数量

