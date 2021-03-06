# Redis 主从复制 （Replication）

Redis主从模式现数据备份

## 主从复制操作机制：

* 当master与slave连接良好，master会向slave发送它接收的命令来保持数据同步
* 当master与slave由于网络等问题断开时，在slave重连后，会尝试获取断开期间的同步命令
* 当部分同步失败时，master会将所有数据创建一个快照版本，一起发送给slave,继续后续同步命令

## 主从复制特性:

* Redis 使用异步复制，slave 和 master 之间异步地确认处理的数据量
* 一个 master 可以拥有多个 slave
* slave可以接收其他多个slave连接,来实现层叠式的主从从结构,像A->B->C,其中B是A的slave,C是B的slave
* Redis复制是异步的,意味着master向slave发送同步命令时,可以继续接收客户端数据
* slave端的复制也是异步的,当 slave 进行初次同步时，它可以使用旧数据集处理查询请求。(replica-serve-stale-data 配置为**yes**时,会正常返回,可能返回空数据集,如果配置为**no**,将返回客户端一个错误)同步之后，旧数据集必须被删除，同时加载新的数据集。 slave 在这个短暂的时间窗口内（如果数据集很大，会持续较长时间），会阻塞到来的连接请求。
* 可以使用复制来避免 master 将全部数据集写入磁盘造成的开销：一种典型的技术是配置你的 master Redis.conf 以避免对磁盘进行持久化，然后连接一个 slave ，其配置为不定期保存或是启用 AOF。但是，这个设置必须小心处理，因为重新启动的 master 程序将从一个空数据集开始：如果一个 slave 试图与它同步，那么这个 slave 也会被清空

## Redis复制实现原理

每一个 Redis master 都有一个 replication ID ：这是一个较大的伪随机字符串，标记了一个给定的数据集。每个 master 也持有一个偏移量，master 将自己产生的复制流发送给 slave 时，发送多少个字节的数据，自身的偏移量就会增加多少，目的是当有新的操作修改自己的数据集时，它可以以此更新 slave 的状态。复制偏移量即使在没有一个 slave 连接到 master 时，也会自增，所以基本上每一对给定的

> Replication ID, offset

都会标识一个 master 数据集的确切版本。

当 slave 连接到 master 时，它们使用 PSYNC 命令来发送它们记录的旧的 master replication ID 和它们至今为止处理的偏移量。通过这种方式， master 能够仅发送 slave 所需的增量部分。但是如果 master 的缓冲区中没有足够的命令积压缓冲记录，或者如果 slave 引用了不再知道的历史记录（replication ID），则会转而进行一个全量重同步：在这种情况下， slave 会得到一个完整的数据集副本，从头开始。

下面是一个全量同步的工作细节：

master 开启一个后台保存进程，以便于生产一个 RDB 文件。同时它开始缓冲所有从客户端接收到的新的写入命令。当后台保存完成时， master 将数据集文件传输给 slave， slave将之保存在磁盘上，然后加载文件到内存。再然后 master 会发送所有缓冲的命令发给 slave。这个过程以指令流的形式完成并且和 Redis 协议本身的格式相同。

## 主从复制配置方案:

需要在所有的slave主机中找到redis.conf进行配置

```properties
# 配置master服务器redis的IP和port
slaveof 192.168.64.1 6379
# master服务器授权密码
masterauth root
# master 服务器redis用户名 如果是default可以不用配置
masteruser default
# slave是否只读,一般配置为yes,防止写入数据导致同步失败
replica-read-only yes
# 是否使用无盘同步,启动无盘同步会带来一定的风险
repl-diskless-sync no
# 无盘同步延迟时间,以等待其他slave就绪
repl-diskless-sync-delay 5
# slave服务器优先级,master down状态时优先升级为master,数字小的排在前列,为0时永不升级为master
replica-priority 100
```

# 哨兵模式

Redis-Sentinel是Redis官方推荐的高可用性(HA)解决方案，当用Redis做Master-slave的高可用方案时，如果master宕机，Redis自身并不能实现自动进行主备切换，sentinel可以监控复制节点的状态，当主节点宕机后，它能根据选举方式选出后端的一个从节点作为新的master，sentinel还能监控多个master-slave集群，发现master宕机后能进行自动切换。

## redis主从设计拓扑图:

<img src="data:image/png;base64,0M8R4KGxGuEAAAAAAAAAAAAAAAAAAAAAPgADAP7/CQAGAAAAAAAAAAAAAAABAAAAFAAAAAAAAAAAEAAA/v///wAAAAD+////AAAAABUAAAD///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////8AOAAAeJztewVUnUuy7sbd3UKAjbu7u0MgSIDg7u6B4O7BCe4El6BBg1twCBosuAQJejc5M5MTbs6smbfWu3e99ab26p/dTVdXd1Xv6q+76x8bRVvJrsJfTZNz/nZmJCQb9s00iaUMGkYAAAG4u4cDQAN+Ehgo4f09gwoA4P6t7O7+/v6hCB+U7v9D/0+RkiwEJPwPgwYrrAn9ydgAOFAysjVUcrC1c2R8qIYJKoD9Ue1tGGGvFsjyduA/5sHPavp2dgxu1lYZscq2fkyYgctJe4KoS9lQPcXFuMGn5nNy7d/NUh0CTcY4rNmvDtWNlM+bGjxHrBXbzmM28q1bWM268jUulUO8USDjyXGmOpex0vR2PD8+U/4kAa9xqc50x+L9ilSLHstssMONQSQ5UFerdBMWT01UVn57utM66lnDJE8xLTBackSPfq4YSl854hiyo8ePKhtxkn/sEGPL9Hp57Fl11KgXlvqGr8ozoVJ8C5LbbbvUyoR68u2DOPp37XlowrdC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AAAAAAAAAAAAAAAAAAAAABYABQD//////////wEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAD+////AAAAAAAAAABfADEAMAAyADUAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAACAf///////////////wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAHJwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAAAAAgAAAAMAAAAEAAAABQAAAAYAAAAHAAAACAAAAAkAAAAKAAAACwAAAAwAAAANAAAADgAAAA8AAAAQAAAAEQAAABIAAAATAAAA/v////7////9/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////w=="/>

功能:

* 监控:  能不断的检测master 和slave是否正常工作
* 通知: 当监控的redis实例发生错误时,可以通过其Api通知系统管理员或其他电脑程序
* 自动故障转移:如果master发生故障时,sentinel会开启一个故障转移进程,将投票挑选出其中一个slave升级为一个新的master,其他slave配置到新的master,形成新的主从复制结构
* 提供配置:客户端可以连接到sentinel来操作redis数据读写,sentinel会自动找到master服务器发送redis命令,当发生故障转移时,Redis系统依然能正常工作

## 启动Redis Sentinel命令

```shell
redis-sentinel /path/to/sentinel.conf
```

或者

```shell
redis-server /path/to/sentinel.conf --sentinel
```

两个命令启动的效果是一样的.

```properties
# 端口
port 26379
# 是否后台启动进程
daemonize yes/no
# master-group-name: master服务器的名字,不许有特殊字符
# ip: master服务器IP
# port: master服务器端口
# quorum: 当sentinel无法连接到master时,会标记sdown,即主观认为是down状态,当sentinel标记的sdown状态超过quonum的数量时,为master标记odown状态,开启故障转移进程
sentinel monitor <master-group-name> <ip> <port> <quorum>
# 主服务器授权码
sentinel auth-pass <master-name> <password>
# 故障转移超时时间,当超过配置的时间,任务故障转移失败
sentinel failover-timeout <master-name> <milliseconds>
# 当sentinel在指定指定时间内无法连接到master,主观任务down状态,标记+sdown
sentinel down-after-milliseconds <master-name> <milliseconds>
# 选项指定了在执行故障转移时， 最多可以有多少个从服务器同时对新的主服务器进行同步
sentinel parallel-syncs <master-name> <numreplicas>

```

## 运行时重新配置

```shell
# 运行时新增监控对象
SENTINEL MONITOR <name> <ip> <port> <quorum>
# 移除监控对象
SENTINEL REMOVE <name>
# 运行时修改master配置 name为master服务器Redis名字 
SENTINEL SET <name> <option> <value>
```

## 添加或移除Sentinel

添加一个Sentinel可以直接配置并启动该进程,由于Sentinel由自动发现的机制,如果想要一次性添加多个Sentinel,需要一个接一个的添加,等待至少30s后其他的Sentinel准备就绪后再添加下一个.

如果想删除一个Sentinel,需要先停止该进程,然后在其他的Sentinel发送`sentinel reset *`命令,*可以替换成mastername

最后在检查一下每个sentinel, 通过命令`SENTINEL MASTER mastername`

## 移除一个旧的master或者不能连接的slave

sentinel不会每一个slave,即使很长时间不能连接,因为它要在slave重新连接时重新配置,所以需要执行`sentinel master mastername`,在接下来的10s会刷新这个配置

