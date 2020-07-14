Redis命令集

redis数据类型

* string(字符串)
* list(列表)
* hash（哈希）
* set（集合）
* zset （有序集合）

# Key Command

## set

```redis
> set mykey somevalue
OK
> get mykey
somevalue
```

set命令选项

* EX: 指定对象过期的时间，以秒为单位
* PX:指定对象的过期时间，以毫秒为单位
* NX:当key不存在的时候设置
* XX:当key存在时添加对象

## exists

```redis
> exists key [key...]
```

* 如果key存在返回1
* 如果key不存在返回0
* 如果指定了多个key返回key存在的数量

## del

`del key [key...]`

删除指定的key,如果key不存在将被忽略，返回被移除的key的数量

## unlink

与del相似，但是非阻塞式/异步操作

## type

`type key`

返回key对应的数据类型 (String,list,hash,set,zset),如果key不存在返回none

## touch

`touch key`

修改key的最后访问时间，返回操作的key的数量

## [keys](https://redis.io/commands/keys)

`keys pattern`

返回匹配的key数组

* h?llo matches hello, hallo and hxllo
* h*llo matches hllo and heeeello
* h[ae]llo matches hello and hallo, but not hillo
* h[^e]llo matches hallo, hbllo, ... but not hello
* h[a-b]llo matches hallo and hbllo

特殊字符使用**\\**转义

## [scan](https://redis.io/commands/scan)

`SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]`

* pattern:匹配的格式
* count:数量
* type:数据类型

SCAN命令是一个基于游标的迭代器（cursor based iterator）： SCAN命令每次被调用之后， 都会向用户返回一个新的游标， 用户在下次迭代时需要使用这个新游标作为 SCAN命令的游标参数， 以此来延续之前的迭代过程。

## [randomKey](https://redis.io/commands/randomkey)

`randomkey key`

在当前数据库中返回随机的key,如果数据库是空的返回`nil`

## [rename](https://redis.io/commands/rename)

`rename key newkey`

重命名key为一个新的key，如果key不存在返回一个错误，如果新的key存在将会被覆盖

## [renamenx](https://redis.io/commands/renamenx)

`renamenx sourcekey targetkey` 

如果key被重命名为新key返回1,如果新key存在返回0,重命名失败

## [expire](https://redis.io/commands/expire)

`expire key seconds`

设置key的生命周期，以秒为单位，如果key存在返回1,不存在返回0

