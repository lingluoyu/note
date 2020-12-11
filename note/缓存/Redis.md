### Redis简介

#### 简介

Redis 是一种基于**键值对（key-value）**的 速度非常快NoSQL 数据库，Redis 中的值可以是**string（字符串）、hash（哈希）、list（列表）、set（集合）、zset（有序集合）、Bitmaps（位图）、HyperLogLog（基数统计）、GEO（地理信息定位）** 等多种数据结构和算法，因此Redis可以满足很多不同的使用场景。Redis多有操作都在内存中，所以读速度非常快，还可以将内存的数据利用快照和日志的形式保存到硬盘上，这样在发生类似断电或者机器故障的时候，内存中的数据不会“丢失”。除了上述功能以外，Redis 还提供了键过期、发布订阅、事务、流水线、Lua 脚本、主从复制等附加功能。

#### 特点

- 读写速度快

  - 基于内存操作
  - C语言实现，一般来说，C 语言实现的程序“距离”操作系统更近，执行速度相对会更快。
  - 基于非阻塞的I/O多路复用机制
  - 单线程，避免了多线程的频繁上下文切换问题，预防了多线程可能产生的竞争问题

- 基于键值对存储

  - key-value映射模式

  - 丰富的数据类型

    > STRING（字符串）、LIST（列表）、SET（集合）、HASH（散列）和ZSET（有序集合）

- 多样化功能

  - 键过期，可以用来实现缓存、分布式锁等。
  - 提供了基础的发布订阅能力，可以用来实现消息系统
  - 支持 Lua 脚本功能
  - 持久化，Redis 本身是基于内存的，所以提供了 AOF 和 RDB 两种方式来持久化。
  - 支持高可用，Redis 集群模式提供了主从复制、哨兵模式和 cluster 模式，当然主从和哨兵一般是配合使用的。

#### 应用场景

- 缓存，可以缓存热点数据（高频读、低频写），分摊数据库压力。
- 排行榜，提供列表和有序集合，可以合理利用这种数据结构实现排行。
- 计数器，比如像统计访问量或者播放量，Redis 天然支持计数功能而且计数的性能也非常好。
- 社交网络，赞/踩、粉丝、共同好友/喜好、推送、下拉刷新等功能。
-  消息队列系统，基于 Redis 发布订阅，可以实现基础的消息队列功能。

#### Redis核心对象

![redisObject对象](https://gitee.com/LoopSup/image/raw/master/img/20191210155928.png)

redis内部使用一个redisObject对象来表示所有的key和value，redisObject最主要的信息如上图所示：type表示一个value对象具体是何种数据类型，encoding是不同数据类型在redis内部的存储方式。比如：type=string表示value存储的是一个普通字符串，那么encoding可以是raw或者int。

### 安装

1. 下载安装包

```bash
wget https://download.redis.io/releases/redis-6.0.9.tar.gz
```

2. 安装

```bash
#解压安装包
tar -zxvf redis-6.0.8.tar.gz 
#安装gcc依赖
yum install gcc
#进入redis解压目录
cd redis-6.0.8
#编译
make
#安装 
make install PREFIX=/usr/local/redis
#验证
redis-cli -v
```

**make时报错**

```bash
                           ^
server.c:2959:48: error: ‘struct redisServer’ has no member named ‘tlsfd’
         if (aeCreateFileEvent(server.el, server.tlsfd[j], AE_READABLE,
```

解决方案：升级gcc版本

```bash
yum -y install centos-release-scl

yum -y install devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-binutils

#scl命令为临时启用
scl enable devtoolset-9 bash

#永久命令
echo "source /opt/rh/devtoolset-9/enable" >>/etc/profile
```

### 数据结构和命令

Redis提供的数据类型主要分为5种自有类型和一种自定义类型，这5种自有类型包括：STRING（字符串）、LIST（列表）、SET（集合）、HASH（散列）和ZSET（有序集合）。

#### 字符串

字符串可以存储一下三种类型的值

- 字符串
- 整数（可自增/increment或自减/decrement）
- 浮点数（可自增/increment或自减/decrement）

**Redis自增和自减命令**

| 命令        | 用例和描述                                                   |
| ----------- | ------------------------------------------------------------ |
| INCR        | INCR key-name--将键值加1                                     |
| DECR        | DECR key-name--将键值减1                                     |
| INCRBY      | INCRBY key-name amount--将键值加上整数amount                 |
| DECRBY      | DECRBY key-name amount--将键值减去整数amount                 |
| INCRBYFLOAT | INCRBYFLOAT key-name amount--将键值加上浮点数amount，Redis2.6或以上版本可用 |

**Redis处理字符串命令**

| 命令     | 用例和描述                                                   |
| -------- | ------------------------------------------------------------ |
| APPEND   | APPEND key-name value--将value追加到给定key-name键的值末尾   |
| GETRANGE | GETRANGE key-name start end--获取key-name的由start至end的字符，包括start和end |
| SETRANGE | SETRANGE key-name offset value--将从offset开始子串设置为value |
| GETBIT   | GETBIT key-name offset--将字符串看做二进制位串（bit string），并返回offset位置的值 |
| SETBIT   | SETBIT key-name offset value--将字符串看做二进制位串（bit string），将offset位置值设为value |
| BITCOUNT | BITCOUNT key-name [start end]--统计二进制位串中值为1的二进制位的数量，若给定start和end，则只统计此范围 |
| BITTOP   | BITTOP operation dest-key key-name [key-name ...]--对一个或多个二进制位串执行包括并（AND）、或（OR)、异或（XOR）、非（NOT)在内的任意一种按位运算操作（bitwiseoperation），并将计算结果存入dest-key |

#### 列表

Redis列表允许用户从序列两端推入或弹出元素，获取列表元素，执行常见列表操作。

**列表常用命令**

| 命令   | 用例和描述                                                   |
| ------ | ------------------------------------------------------------ |
| RPUSH  | RPUSH key-name value [value ...]--将一个或多个值推入列表右端 |
| LPUSH  | LPUSH key-name value [value ...]--将一个或多个值推入列表左端 |
| RPOP   | RPOP key-name--移除并返回列表最右端元素                      |
| LPOP   | LPOP key-name--移除并返回列表最左端元素                      |
| LINDEX | LINDEX key-name offset--返回列表中偏移量为offset的元素       |
| LRANGE | LRANGE key-name start end--返回列表start至end内的所有元素，包含start和end |
| LTRIM  | LTRIM key-name start end--对列表进行修剪，只保留start至end元素，包含start和end |

**列表操作命令**

| 命令       | 用例和描述                                                   |
| ---------- | ------------------------------------------------------------ |
| BLPOP      | BLPOP key-name [key-name ...] timeout--从第一个非空列表中弹出位于最左端的元素，或在timeout秒之内阻塞并等待可弹出元素出现 |
| BRPOP      | BRPOP key-name [key-name ...] timeout--从第一个非空列表中弹出位于最右端的元素，或在timeout秒之内阻塞并等待可弹出元素出现 |
| RPOPLPUSH  | RPOPLPUSH source-key dest-key--从source-key列表中弹出最右端元素，并推入dest-key列表最左端，并返回此元素 |
| BRPOPLPUSH | BRPOPLPUSH source-key dest-key timeout--从source-key列表中弹出最右端元素，并推入dest-key列表最左端，并返回此元素；若source-key为空，在timeout秒内阻塞并等待可弹出元素出现 |

#### 集合

Redis集合以无序的方式存储多个各不相同的元素，用户可以对集合执行添加元素、移除元素操作以及检查一个元素是否存在于集合里。

**集合常用命令**

| 命令        | 用例和描述                                                   |
| ----------- | ------------------------------------------------------------ |
| SADD        | SADD key-name item [item ...]--将一个或多个元素添加到集合中，并返回被添加元素中原本不存在于集合里元素数量 |
| SREM        | SREM key-name item [item ...]--从集合里移除一个或多个元素，并返回被移除元素数量 |
| SISMEMBER   | SISMEMBER key-name item--检查item是否存在于集合key-name里    |
| SCARD       | SCARD key-name--返回集合包含的元素的数量                     |
| SMEMBERS    | SMEMBERS key-name--返回集合包含的所有元素                    |
| SRANDMEMBER | SRANDMEMBER key-name [count]--从集合中随机返回一个或多个元素。当count为正数时，命令返回的随机元素不会重复；count为负数是，命令返回的随机元素可能会出现重复。 |
| SPOP        | SPOP key-name--随机移除一个元素，并返回被移除的元素          |
| SMOVE       | SMOVE source-key dest-key item--若集合source-key包含item，从集合source-key里移除item，并将item添加到dest-key中；若item被成功移除，返回1，否则返回0 |

**组合和处理多个集合命令**

| 命令        | 用例和描述                                                   |
| ----------- | ------------------------------------------------------------ |
| SDIFF       | SDIFF key-name [key-name...]--返回存在于第一个集合、但不存在于其他集合的元素（差集） |
| SDIFFSTORE  | SDIFFSTORE dest-key key-name [key-name ...]--将存在于第一个集合、但不存在于其他集合的元素（差集）存储到dest-key |
| SINTER      | SINTER key-name [key-name...]--返回同时存在于所有集合中的元素（交集） |
| SINTERSTORE | SINTERSTORE dest-key key-name [key-name ...]--将同时存在于所有集合中的元素（交集）存储到dest-key |
| SUNION      | SUNION key-name [key-name...]--返回至少存在于一个集合中的元素（并集） |
| SUNIONSTORE | SUNIONSTORE dest-key key-name [key-name ...]--将至少存在于一个集合中的元素（并集）存储到dest-key |

#### 散列

散列可以让用户将多个键值对存储到一个Redis键中。

适用于将一些相关的数据存储在一起。

**添加和删除键值对命令**

| 命令  | 用例和描述                                                   |
| ----- | ------------------------------------------------------------ |
| HMGET | HMGET key-name key [key ...]--从散列中获取一个或多个键值     |
| HMSET | HMSET key-name key value [key value ...]--为散列中一个或多个键设置值 |
| HDEL  | HDEL key-name key [key ...]--删除散列中一个或多个键值对，返回成功找到并删除的键值对数量 |
| HLEN  | HLEN key-name--返回散列包含键值对数量                        |

**散列高级命令**

| 命令         | 用例和描述                                                   |
| ------------ | ------------------------------------------------------------ |
| HEXISTS      | HEXISTS key-name key--检查给定键是否存在于散列中             |
| HKEYS        | HKEYS key-name--获取散列包含的所有键                         |
| HVALS        | HVALS key-name--获取散列包含的所有值                         |
| HGETALL      | HGETALL key-name--获取散列包含的所有键值对                   |
| HINCRBY      | HINCRBY key-name key increment--将键key存储的值加上整数increment |
| HINCRBYFLOAT | HINCRBYFLOAT key-name key increment--将键key存储的值加上浮点数increment |

#### 有序集合

和散列存储着键与值之间的映射类似，有序集合也存储着成员与分值之间的映射，并提供了分值处理命令，以及根据分值大小有序的获取（fetch）或扫描（scan）成员和分值的命令。

**有序集合常用命令**

| 命令    | 用例和描述                                                   |
| ------- | ------------------------------------------------------------ |
| ZADD    | ZADD key-name score member [score member ...]--将带有给定分值的成员添加到有序集合里 |
| ZREM    | ZREM key-name member [member ...]--从有序集合里移除给定成员，并返回被移除成员数量 |
| ZCARD   | ZCARD key-name--获取有序集合包含的成员数量                   |
| ZINCRBY | ZINCRBY key-name increment member--将member成员的分值加上increment |
| ZCOUNT  | ZCOUNT key-name min max--返回分值介于min和max之间的成员数量  |
| ZRANK   | ZRANK key-name member--返回member在有序集合中的排名          |
| ZSCORE  | ZSCORE key-name member--返回成员member的分值                 |
| ZRANGE  | ZRANGE key-name start stop [WITHSCORES]--返回有序集合中排名介于start和stop之间的成员，若给定了可选的WITHSCORES选项，命令会将成员的分值一并返回 |

**有序集合高级命令**

| 命令             | 用例和描述                                                   |
| ---------------- | ------------------------------------------------------------ |
| ZREVRANK         | ZREVRANK key-name member--返回有序集合里成员member的排名，成员按照分值从大到小排列 |
| ZREVRANGE        | ZREVRANGE key-name start stop [WITHSCORES]--返回有序集合中给定排名范围内的成员，成员按照分值从大到小排列 |
| ZRANGEBYSCORE    | ZRANGEBYSCORE key-name min max [WITHSCORES] [LINMIT offset count]--返回有序集合中，分值介于min和max之间的所有成员 |
| ZREVRANGEBYSCORE | ZREVRANGEBYSCORE key-name min max [WITHSCORES] [LINMIT offset count]--返回有序集合中，分值介于min和max之间的所有成员，并按照分值从大到小排列 |
| ZREMRANGEBYRANK  | ZREMRANGEBYRANK key-name start stop--移除有序集合中排名介于min和max之间所有成员 |
| ZREMRANGEBYSCORE | ZREMRANGEBYSCORE key-name start stop--移除有序集合中分值介于min和max之间所有成员 |
| ZINITERSTORE     | ZINITERSTORE dest-key key-count key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM\|MIN\|MAX]--对给定有序集合进行类似于集合的交集预算 |
| ZUNIONSTORE      | ZUNIONSTORE dest-key key-count key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM\|MIN\|MAX]--对给定有序集合进行类似于集合的并集预算 |

#### 发布订阅

发布与订阅（pub/sup）特点是订阅者（listener）负责订阅频道（channel），发送者（publisher）负责向频道发送二进制字符串消息（binary string message）。

**发布与订阅命令**

| 命令         | 用例和描述                                                   |
| ------------ | ------------------------------------------------------------ |
| SUBSCRIBE    | SUBSCRIBE channel [channel ...]--订阅给定的一个或多个频道    |
| UNSUBSCRIBE  | UNSUBSCRIBE channel [channel ...]--退订给定的一个或多个频道，若执行时未给定任何频道，则退订所有频道 |
| PUBLISH      | PUBLISH channel message--向给定频道发消息                    |
| PSUBSCRIBE   | PSUBSCRIBE pattern [pattern ...]--订阅与给定模式相匹配的所有频道 |
| PUNSUBSCRIBE | PUNSUBSCRIBE pattern [pattern ...]--退订给定模式的频道，若执行时未给定任何模式，则退订所有模式 |

#### 其他命令

**排序**

SORT命令可以对列表、集合及有序集合进行排序。

| 命令 | 用例和描述                                                   |
| ---- | ------------------------------------------------------------ |
| SORT | SORT source-key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC\|DESC] [ALPHA] [STORE dest-key]--根据给定选项，对输入列表、集合或有序集合进行排序，然后返回或存储排序结果 |

**基本事务**

Redis事务用到`MULTI`命令和`EXEC`命令，这种事务可以让一个客户端在不被其他客户端打断的情况下执行多个命令。

```shell
MULTI
...
EXEC
```

**键过期时间**

Redis可以通过过期时间（expiration）特性来让一个键在给定的时限（timeout）之后被自动删除。

| 命令      | 用例和描述                                                   |
| --------- | ------------------------------------------------------------ |
| PERSIST   | PERSIST key-name--移除键的过期时间                           |
| TTL       | TTL key-name--查看给定键距离过期还有多少秒                   |
| EXPIRE    | EXPIRE key-name seconds--让给定键在制定秒数之后过期          |
| EXPIREAT  | EXPIREAT key-name timestamp--将给定键的过期时间设置为给定UNIX时间戳 |
| PTTL      | PTTL key-name--查看给定键距离过期时间还有多少毫秒，Redis2.6或以上版本可用 |
| PEXPIRE   | PEXPIRE key-name milliseconds--让给定键在指定毫秒数之后过期，Redis2.6或以上版本可用 |
| PEXPIREAT | PEXPIREAT key-name timestamp-milliseconds--将一个毫秒级精度的UNIX时间戳设置为给定键的过期时间，Redis2.6或以上版本可用 |

### Redis持久化

Redis提供了两种不同的持久化方法将数据存储到硬盘。

- 快照（snapshotting，RDB），将存在于某一时刻的所有数据都写入硬盘。
- 只追加文件（append-only file，AOF），在执行命令时，将被执行的写命令复制到硬盘里。

**redis.conf持久化配置**

- RDB

```bash
#是否开启rdb压缩 默认开启
rdbcompression yes
#代表900秒内有一次写入操作，就记录到rdb
save 900 1
# rdb的备份文件名称
dbfilename dump.rdb
# 表示备份文件存放位置
dir ./
```

- AOF

```bash
# 是否开启aof，默认是关闭的
appendonly no
#aof的文件名称
appendfilename "appendonly.aof"
# no: don't fsync, just let the OS flush the data when it wants. Faster.
# always: fsync after every write to the append only log. Slow, Safest.
# everysec: fsync only one time every second. Compromise.
appendfsync everysec
# 在进行rewrite的时候不开启fsync，即不写入缓冲区，直接写入磁盘，这样会造成IO阻塞，但是最为安全，如果为yes表示写入缓冲区，写入的适合redis宕机会造成数据持久化问题(在linux的操作系统的默认设置下，最多会丢失30s的数据)
no-appendfsync-on-rewrite no
# 下面两个参数要配合使用，代表当redis内容大于64m同时扩容超过100%的时候会执行bgrewrite，进行持久化
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

#### RDB

**优缺点**

- RDB 会生成多个数据文件，每个数据文件都代表了某一个时刻中 Redis 的数据，这种多个数据文件的方式，非常适合做冷备，可以将这种完整的数据文件发送到一些远程的安全存储上去，比如说 Amazon 的 S3 云服务上去，在国内可以是阿里云的 ODPS 分布式存储上，以预定好的备份策略来定期备份 Redis 中的数据。
- RDB 对 Redis 对外提供的读写服务，影响非常小，可以让 Redis 保持高性能，因为 Redis 主进程只需要 fork 一个子进程，让子进程执行磁盘 IO 操作来进行 RDB 持久化即可。
- 相对于 AOF 持久化机制来说，直接基于 RDB 数据文件来重启和恢复 Redis 进程，更加快速。
- 如果想要在 Redis 故障时，尽可能少的丢失数据，那么 RDB 没有 AOF 好。一般来说，RDB数据快照文件，都是每隔 5 分钟，或者更长时间生成一次，这个时候就得接受一旦 Redis 进程宕机，那么会丢失最近 5 分钟的数据。
- RDB 每次在 fork 子进程来执行 RDB 快照数据文件生成的时候，如果数据文件特别大，可能会导致对客户端提供的服务暂停数毫秒，或者甚至数秒。

**创建快照的几种方法**

- 客户端向redis发送`BGSAVE`的命令（注意windows不支持`BGSAVE`），此时reids调用 **fork** 创建子进程，子进程将快照写入磁盘，父进程继续处理命令请求。
- 客户端发送`SAVE`命令创建快照。注意这种方式会**阻塞**整个父进程。很少使用，特殊情况（没有足够内存去执行`BGSAVE`命令）才使用。
- 用户设置了`save`配置选项，当配置的条件被满足时，Redis会自动触发一次`BGSAVE`命令。
- redis通过`SHUTDOWN`命令关闭服务器请求的时候，此时redis会停下所有工作执行一次save，阻塞所有客户端不再执行任何命令并且进行磁盘写入，写入完成关闭服务器。
- redis集群的时候，会发送`SYNC`命令进行一次复制操作，如果主服务器**没有执行**或者**并非刚刚执行完**`BGSAVE`，则会进行`BGSAVE`。
- 执行**flushall** 命令。

#### AOF

**优缺点**

- AOF 可以更好的保护数据不丢失，一般 AOF 会每隔 1 秒，通过一个后台线程执行一次fsync 操作，最多丢失 1 秒钟的数据。
- AOF 日志文件以 append-only 模式写入，所以没有任何磁盘寻址的开销，写入性能非常高，而且文件不容易破损，即使文件尾部破损，也很容易修复。
- AOF 日志文件即使过大的时候，出现后台重写操作，也不会影响客户端的读写。因为在rewrite log 的时候，会对其中的指令进行压缩，创建出一份需要恢复数据的最小日志出来。在创建新日志文件的时候，老的日志文件还是照常写入。当新的 merge 后的日志文件ready 的时候，再交换新老日志文件即可。
- AOF 日志文件的命令通过可读较强的方式进行记录，这个特性非常适合做灾难性的误删除的紧急恢复。比如某人不小心用 flushall 命令清空了所有数据，只要这个时候后台rewrite 还没有发生，那么就可以立即拷贝 AOF 文件，将最后一条 flushall 命令给删了，然后再将该 AOF 文件放回去，就可以通过恢复机制，自动恢复所有数据。
- 对于同一份数据来说，AOF 日志文件通常比 RDB 数据快照文件更大。
- AOF 开启后，支持的写 QPS 会比 RDB 支持的写 QPS 低，因为 AOF 一般会配置成每秒fsync 一次日志文件，当然，每秒一次 fsync ，性能也还是很高的。（如果实时写入，那么 QPS 会大降，Redis 性能会大大降低）
- 以前 AOF 发生过 bug，就是通过 AOF 记录的日志，进行数据恢复的时候，没有恢复一模一样的数据出来。所以说，类似 AOF 这种较为复杂的基于命令日志 / merge / 回放的方式，比基于 RDB 每次持久化一份完整的数据快照文件的方式，更加脆弱一些，容易有 bug。不过AOF 就是为了避免 rewrite 过程导致的 bug，因此每次 rewrite 并不是基于旧的指令日志进行merge 的，而是基于当时内存中的数据进行指令的重新构建，这样健壮性会好很多。

AOF持久化配置通过`appendonly yes`打开，通过`appendfsync`配置AOF文件同步频率。

**文件同步** file.write()-->缓冲区-->硬盘

**appendfsync选项**

| 选项     | 同步频率                                             |
| -------- | ---------------------------------------------------- |
| always   | 每个Redis写命令都要同步写入磁盘。会严重降低Redis速度 |
| ererysec | 每秒执行一次同步，显式的将多个写命令同步到硬盘       |
| no       | 让操作系统决定何时进行同步                           |

**AOF文件体积过大**

`BGREWRITEAOF`命令，移除AOF文件中冗余命令重写AOF文件，缩小文件体积。

**原理**：Redis创建一个子线程，由子线程负责对AOF文件重写。

| 选项                        | 重写条件                                            |
| --------------------------- | --------------------------------------------------- |
| auto-aof-rewrite-percentage | AOF文件体积比上一次重写之后体积大了至少百分之？重写 |
| auto-aof-rewrite-min-size   | AOF体积大小超过某个值重写                           |

#### RDB 和 AOF 到底该如何选择

- 不要仅仅使用 RDB，因为那样会导致你丢失很多数据；
- 也不要仅仅使用 AOF，因为那样有两个问题：第一，你通过 AOF 做冷备，没有 RDB 做冷备来的恢复速度更快；第二，RDB 每次简单粗暴生成数据快照，更加健壮，可以避免 AOF 这种复杂的备份和恢复机制的 bug；
- Redis 支持同时开启开启两种持久化方式，我们可以综合使用 AOF 和 RDB 两种持久化机制，用 AOF 来保证数据不丢失，作为数据恢复的第一选择; 用 RDB 来做不同程度的冷备，在 AOF 文件都丢失或损坏不可用的时候，还可以使用 RDB 来进行快速的数据恢复。

### Redis过期

#### Redis过期策略

Redis 过期策略是：**定期删除+惰性删除**。

**定期删除**：Redis 默认是每隔 100ms 就随机抽取一些设置了过期时间的 key，检查其是否过期，如果过期就删除。

**惰性删除**：获取某个 key 的时候，Redis 会检查一下 ，这个 key 如果设置了过期时间那么是否过期了？如果过期了此时就会删除，不会给你返回任何东西。

如果定期删除漏掉了很多过期 key，然后你也没及时去查，也就
没走惰性删除，就会走内存淘汰机制

**内存淘汰机制**

- noeviction: 当内存不足以容纳新写入数据时，新写入操作会报错，这个一般没人用吧，实在是太恶心了。
- allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）。
- allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个 key，这个一般没人用吧，为啥要随机，肯定是把最近最少使用的 key 给干掉啊。
- volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 key（这个一般不太合适）。
- volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 key。
- volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 key 优先移除。

### Redis集群

#### Redis主从架构

一主多从，主负责写，并将数据复制到slave节点，从节点负责读。![advanced-java](https://gitee.com/LoopSup/image/raw/master/img/20201208095407.jpg)

**Redis replication核心机制**

- Redis 采用异步方式复制数据到 slave 节点，不过 Redis2.8 开始，slave node 会周期性地确认自己每次复制的数据量；
- 一个 master node 是可以配置多个 slave node 的；
- slave node 也可以连接其他的 slave node；
- slave node 做复制的时候，不会 block master node 的正常工作；
- slave node 在做复制的时候，也不会 block 对自己的查询操作，它会用旧的数据集来提供服务；但是复制完成的时候，需要删除旧数据集，加载新数据集，这个时候就会暂停对外服务了；
- slave node 主要用来进行横向扩容，做读写分离，扩容的 slave node 可以提高读的吞吐量。

**主从复制原理**

当启动一个 slave node 的时候，它会发送一个 PSYNC 命令给 master node。

如果这是 slave node 初次连接到 master node，那么会触发一次 full resynchronization 全量复制。此时 master 会启动一个后台线程，开始生成一份 RDB 快照文件，同时还会将从客户端 client 新收到的所有写命令缓存在内存中。 RDB 文件生成完毕后， master 会将这个 RDB 发送给 slave，slave 会先写入本地磁盘，然后再从本地磁盘加载到内存中，接着 master 会将内存中缓存的写命令发送到 slave，slave 也会同步这些数据。slave node 如果跟 master node 有
网络故障，断开了连接，会自动重连，连接之后 master node 仅会复制给 slave 部分缺少的数据。![advanced-java2](https://gitee.com/LoopSup/image/raw/master/img/20201208100724.jpg)

#### Redis故障转移

sentinel（哨兵），主要功能

- 集群监控：负责监控 Redis master 和 slave 进程是否正常工作。
- 消息通知：如果某个 Redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
- 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。
- 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。

哨兵用于实现 Redis 集群的高可用，本身也是分布式的，作为一个哨兵集群去运行，互相协同工作。

- 故障转移时，判断一个 master node 是否宕机了，需要大部分的哨兵都同意才行，涉及到了分布式选举的问题。
- 即使部分哨兵节点挂掉了，哨兵集群还是能正常工作的，因为如果一个作为高可用机制重要组成部分的故障转移系统本身是单点的，那就很坑爹了。

### Redis雪崩、穿透和击穿

### 雪崩

Redis缓存层由于某种原因宕机后，所有的请求会涌向存储层（数据库），短时间内的高并发请求可能会导致存储层挂机，称之为“Redis雪崩”。

![缓存雪崩](https://gitee.com/LoopSup/image/raw/master/img/20191210143147.png)

缓存雪崩的事前事中事后的**解决方案**如下：

- 事前：Redis 高可用，主从+哨兵，Redis cluster，避免全盘崩溃。
- 事中：本地 ehcache 缓存 + hystrix 限流&降级，避免 MySQL 被打死。
- 事后：Redis 持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。

#### 穿透

请求在数据库中不存在的数据，Redis缓存无法查询到数据，因此无法缓存，每次请求都会到数据库。

![缓存穿透](https://gitee.com/LoopSup/image/raw/master/img/20191210143331.png)

**解决方案**

每次系统从数据库中只要没查到，就写一个空值到缓存里去，比如 set
-999 UNKNOWN 。然后设置一个过期时间，这样的话，下次有相同的 key 来访问的时候，在缓存失效之前，都可以直接从缓存中取数据。

#### 击穿

某个 key 非常热点，访问非常频繁，处于集中式高并发访问的情况，当这个
key 在失效的瞬间，大量的请求就击穿了缓存，直接请求数据库。

**解决方案**

- 若缓存的数据是基本不会发生更新的，则可尝试将该热点数据设置为永不过期。
- 若缓存的数据更新不频繁，且缓存刷新的整个流程耗时较少的情况下，则可以采用基于Redis、zookeeper 等分布式中间件的分布式互斥锁，或者本地互斥锁以保证仅少量的请求能请求数据库并重新构建缓存，其余线程则在锁释放后能访问到新缓存。
- 若缓存的数据更新频繁或者在缓存刷新的流程耗时较长的情况下，可以利用定时线程在缓存过期前主动地重新构建缓存或者延后缓存的过期时间，以保证所有的请求能一直访问到对应的缓存。

### 布隆过滤器（Bloom Filter）

* Bloom filter 是由 Howard Bloom 在 1970 年提出的二进制向量数据结构，它具有很好的空间和时间效率，被用来检测一个元素是不是集合中的一个成员。如果检测结果为是，该元素不一定在集合中；但如果检测结果为否，该元素一定不在集合中。因此Bloom filter具有100%的召回率。这样每个检测请求返回有“在集合内（可能错误）”和“不在集合内（绝对不在集合内）”两种情况，可见 Bloom filter 是牺牲了正确率和时间以节省空间。

* 海量数据查重

![海量数据查重](https://gitee.com/LoopSup/image/raw/master/img/20191210143246.png)

* 避免缓存穿透

![避免缓存穿透](https://gitee.com/LoopSup/image/raw/master/img/20191210143218.png)

