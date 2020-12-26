## Redis集群

Redis 集群有三种模式

- 主从模式
- Sentinel模式
- Cluster模式

### 主从模式

主从复制模式中，有两种数据库：主数据库(master)和从数据库(slave)。

- 主数据库可以进行读写操作，当读写操作导致数据变化时会自动将数据同步给从数据库
- 从数据库一般都是只读的，并且接收主数据库同步过来的数据
- 一个master可以拥有多个slave，但是一个slave只能对应一个master
- slave挂了不影响其他slave的读和master的读和写，重新启动后会将数据从master同步过来
- master挂了以后，不影响slave的读，但redis不再提供写服务，master重启后redis将重新对外提供写服务
- master挂了以后，不会在slave节点中重新选一个master

#### 工作机制

1. 从数据库向主数据库发送sync(数据同步)命令。
2. 主数据库接收同步命令后，会保存快照，创建一个RDB文件。
3. 当主数据库执行完保持快照后，会向从数据库发送RDB文件，而从数据库会接收并载入该文件。
4. 主数据库将缓冲区的所有写命令发给从服务器执行。
5. 以上处理完之后，之后主数据库每执行一个写命令，都会将被执行的写命令发送给从数据库。

在Redis2.8之后，主从断开重连后会根据断开之前最新的命令偏移量进行增量复制。

![replication](https://gitee.com/LoopSup/image/raw/master/img/20201208100724.jpg)

#### 缺点

master节点在主从模式中唯一，若master挂掉，则redis无法对外提供写服务。

### Sentinel模式

哨兵主要作用是监控redis集群的运行状况，解决了主从复制出现故障时需要人为干预的问题。

- sentinel模式是建立在主从模式的基础上，如果只有一个Redis节点，sentinel就没有任何意义
- 当master挂了以后，sentinel会在slave中选择一个做为master，并修改它们的配置文件，其他slave的配置文件也会被修改，比如slaveof属性会指向新的master
- 当master重新启动后，它将不再是master而是做为slave接收新的master的同步数据
- sentinel因为也是一个进程有挂掉的可能，所以sentinel也会启动多个形成一个sentinel集群
- 多sentinel配置的时候，sentinel之间也会自动监控
- 当主从模式配置密码时，sentinel也会同步将配置信息修改到配置文件中，不需要担心
- 一个sentinel或sentinel集群可以管理多个主从Redis，多个sentinel也可以监控同一个redis
- sentinel最好不要和Redis部署在同一台机器，不然Redis的服务器挂了以后，sentinel也挂了

#### 哨兵主要功能

- 集群监控：负责监控 Redis master 和 slave 进程是否正常工作。
- 消息通知：如果某个 Redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
- 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。
- 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。

#### 主观下线和客观下线

- 主观下线（Subjectively Down， 简称 SDOWN）指的是单个 Sentinel 实例对服务器做出的下线判断。
- 客观下线（Objectively Down， 简称 ODOWN）指的是多个 Sentinel 实例在对同一个服务器做出 SDOWN 判断， 并且通过 SENTINEL is-master-down-by-addr 命令互相交流之后， 得出的服务器下线判断。

#### 原理

当主节点出现故障时，由Redis Sentinel自动完成故障发现和转移，并通知应用方，实现高可用性。

- 每个sentinel以每秒钟一次的频率向它所知的master，slave以及其他sentinel实例发送一个 PING 命令
- 如果**一个实例距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 选项所指定的值**， 则这个实例会被sentinel标记为**主观下线**。 一个有效回复可以是： +PONG 、 -LOADING 或者 -MASTERDOWN 。
- 如果一个master被标记为主观下线，则正在监视这个master的所有sentinel要以每秒一次的频率确认master的确进入了主观下线状态
- 当有**足够数量的sentinel（大于等于配置文件指定的值）在指定的时间范围内确认master的确进入了主观下线状态**， 则master会被标记为**客观下线**
- 在一般情况下， 每个sentinel会以每 10 秒一次的频率向它已知的所有master，slave发送 INFO 命令
- 当master被sentinel标记为客观下线时，sentinel向下线的master的所有slave发送 INFO 命令的频率会从 10 秒一次改为 1 秒一次
- 若没有足够数量的sentinel同意master已经下线，master的客观下线状态就会被移除；若master重新向sentinel的 PING 命令返回有效回复，master的主观下线状态就会被移除

当使用sentinel模式的时候，客户端就不要直接连接Redis，而是连接sentinel的ip和port，由sentinel来提供具体的可提供服务的Redis实现，这样当master节点挂掉以后，sentinel就会感知并将新的master节点提供给使用者。

### Cluster模式

为了解决单机Redis容量有限的问题，将数据按一定的规则分配到多台机器，内存/QPS不受限于单机，可受益于分布式集群高扩展性。Cluster是去中心化的集群，就是各个节点之间都是**平等的**,而且每个节点具有高度自治的特征,节点之间相互连接通讯保持数据的一致性。我们最熟知的去中心应用比特币系统,就是一个去中心化的系统。

- 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽。
- 节点的fail是通过集群中超过半数的节点检测失效时才生效。
- 客户端与redis节点直连，不需要中间代理层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可。
- cluster把所有的物理节点映射到slot上（不一定是平均分配），cluster 负责维护node<->slot<->value。
- Redis集群预分好16384个桶，当需要在 Redis 集群中放置一个 key-value 时，根据 CRC16(key) mod 16384的值，决定将一个key放到哪个桶中。

#### 工作方式

##### Redis集群数据分片（Redis Cluster data sharding）

Redis集群中有16384个hash slots，为了计算给定的key应该在哪个hash slot上，我们简单地用这个key的CRC16值来对16384取模（即：key的CRC16  %  16384）。

```
HASH_SLOT = CRC16(key) mod 16384
```

Redis集群中的每个节点负责一部分hash slots，假设你的集群有3个节点，那么：

- 节点 A 包含 0 到 5500号哈希槽.
- 节点 B 包含5501 到 11000 号哈希槽.
- 节点 C 包含11001 到 16384号哈希槽.

允许添加和删除集群节点。比如，如果你想增加一个新的节点D，那么久需要从A、B、C节点上删除一些hash slot给到D。同样地，如果你想从集群中删除节点A，那么会将A上面的hash slots移动到B和C，当节点A上是空的时候就可以将其从集群中完全删除。

因为将hash slots从一个节点移动到另一个节点并不需要停止其它的操作，添加、删除节点以及更改节点所维护的hash slots的百分比都不需要任何停机时间。也就是说，移动hash slots是并行的，移动hash slots不会影响其它操作。

Redis支持多个key操作，只要这些key在一个单个命令中执行（或者一个事务，或者Lua脚本执行），那么它们就属于相同的hash slot。你也可以用hash tags强制多个key都在相同的hash slot中。

##### Redis集群主从模式（Redis Cluster master-slave model）

redis cluster 为了保证数据的高可用性，加入了主从模式，一个主节点对应一个或多个从节点，主节点提供数据存取，从节点则是从主节点拉取数据备份，当这个主节点挂掉后，就会有这个从节点选取一个来充当主节点，从而保证集群不会挂掉。

##### Redis 一致性保证（Redis Cluster consistency guarantees）

Redis 并不能保证数据的**强一致性**. 这意味这在实际中集群在特定的条件下可能会丢失写操作.

第一个原因是因为集群是用了异步复制. 写操作过程:

- 客户端向主节点B写入一条命令.
- 主节点B向客户端回复命令状态.
- 主节点将写操作复制给他得从节点 B1, B2 和 B3.

主节点对命令的复制工作发生在返回命令回复之后， 因为如果每次处理命令请求都需要等待复制操作完成的话， 那么主节点处理命令请求的速度将极大地降低 —— 我们必须在性能和一致性之间做出权衡。 注意：Redis 集群可能会在将来提供同步写的方法。 Redis 集群另外一种可能会丢失命令的情况是集群出现了网络分区， 并且一个客户端与至少包括一个主节点在内的少数实例被孤立。

举个例子 假设集群包含 A 、 B 、 C 、 A1 、 B1 、 C1 六个节点， 其中 A 、B 、C 为主节点， A1 、B1 、C1 为A，B，C的从节点， 还有一个客户端 Z1 假设集群中发生网络分区，那么集群可能会分为两方，大部分的一方包含节点 A 、C 、A1 、B1 和 C1 ，小部分的一方则包含节点 B 和客户端 Z1 .

Z1仍然能够向主节点B中写入, 如果网络分区发生时间较短,那么集群将会继续正常运作,如果分区的时间足够让大部分的一方将B1选举为新的master，那么Z1写入B中得数据便丢失了.

注意， 在网络分裂出现期间， 客户端 Z1 可以向主节点 B 发送写命令的最大时间是有限制的， 这一时间限制称为节点超时时间（node timeout）， 是 Redis 集群的一个重要的配置选项